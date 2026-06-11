# Full System Design — E-Commerce Marketplace Platform

## 0. Project Overview

**Project name:** E-Commerce Marketplace Platform  
**Domain:** Marketplace thương mại điện tử Shopee-like cho thị trường Việt Nam  
**Model:** Multi-seller B2C  
**Users:** Buyer, Seller, Admin  
**External systems:** Momo, VNPay, GHN, GHTK  
**Main goal:** Thiết kế một hệ thống marketplace có khả năng xử lý traffic lớn, đặc biệt trong flash sale, đồng thời đảm bảo reliability, scalability, consistency phù hợp cho payment/order và catalog/search.

Hệ thống không chỉ là website bán hàng thông thường, mà là một nền tảng marketplace có nhiều seller, nhiều sản phẩm, nhiều đơn hàng, nhiều phương thức thanh toán và vận chuyển. Vì vậy, thiết kế cần tách các service theo domain, dùng polyglot persistence, caching, message queue, replication, sharding, CDN và monitoring.

---

## 1. Business Problem Definition

### 1.1 Business Context

Thị trường thương mại điện tử Việt Nam có đặc điểm:

- Lượng người dùng lớn, tập trung ở mobile.
- Traffic biến động mạnh theo chiến dịch khuyến mãi, flash sale, ngày đôi như 6/6, 9/9, 11/11.
- Người mua cần tìm sản phẩm nhanh, giá tốt, thanh toán an toàn.
- Seller cần quản lý sản phẩm, tồn kho, đơn hàng, doanh thu.
- Admin cần kiểm duyệt, xử lý khiếu nại, quản lý seller và sản phẩm.
- Hệ thống cần tích hợp nhiều bên thứ ba: payment provider, shipping provider.

### 1.2 Problem Statement

Người mua Việt Nam cần một nền tảng mua sắm online đáng tin cậy. Tuy nhiên, trong các sự kiện flash sale, hệ thống có thể gặp các vấn đề:

- Crash khi traffic tăng đột biến.
- Tìm kiếm sản phẩm chậm.
- Tồn kho không đồng bộ.
- Payment và order có thể bị inconsistent.
- External API như Momo/VNPay/GHN timeout.
- Một lỗi ở một component có thể kéo cả hệ thống xuống.

Mục tiêu của thiết kế là xây dựng hệ thống có thể xử lý **10 lần traffic bình thường** mà không downtime đáng kể, đồng thời giữ data quan trọng như payment/order ở trạng thái consistent.

---

## 2. Business Goals

| Goal | Description | Success Metric |
|---|---|---|
| High availability | Hệ thống luôn sẵn sàng cho buyer/seller | 99.9% uptime |
| Flash sale readiness | Xử lý traffic tăng đột biến | 5K RPS → 50K RPS |
| Fast search | Tìm sản phẩm nhanh | Search P95 < 500ms |
| Fast API | API thông thường phản hồi nhanh | API P95 < 200ms |
| Reliable payment | Không mất tiền, không tạo payment sai | Strong consistency cho payment |
| Inventory correctness | Tránh overselling | Stock reservation + async sync |
| Marketplace scalability | Hỗ trợ nhiều seller/product | 100M products |
| Observability | Phát hiện lỗi nhanh | Grafana/Kafka metrics/logs |

---

## 3. Actors and User Groups

| Actor | Số lượng ước tính | Hành vi chính |
|---|---:|---|
| Buyer | 10M MAU, 2M DAU | Search, browse, add cart, checkout, pay, track order |
| Seller | 500,000 | List product, manage inventory, process order, view dashboard |
| Admin | ~1,000 | Moderate product, manage seller, support, monitor reports |
| External Payment | Momo, VNPay | Process payment, send webhook |
| External Shipping | GHN, GHTK | Create shipment, update delivery status |

---

## 4. Functional Requirements

### 4.1 User Service

- Register account.
- Login/logout.
- JWT authentication.
- Manage profile.
- Manage address.
- Role-based access: buyer, seller, admin.

### 4.2 Product Catalog Service

- Seller creates product.
- Seller updates product information.
- Buyer browses categories.
- Buyer views product detail.
- Product supports images, variants, attributes, tags.
- Product status: draft, active, disabled, moderated.

### 4.3 Search Service

- Search by keyword.
- Filter by category, price, rating, seller.
- Sort by relevance, price, newest, sold count.
- Use Elasticsearch index for fast full-text search.

### 4.4 Cart Service

- Add item to cart.
- Remove item from cart.
- Update quantity.
- Calculate subtotal.
- Validate product availability.
- Cart stored in Redis with TTL.

### 4.5 Inventory Service

- Reserve stock when checkout starts.
- Confirm stock after payment.
- Release stock if payment fails or expires.
- Decrease stock after successful payment.
- Sync inventory event to catalog/search.

### 4.6 Order Service

- Create order.
- Track order status.
- View order history.
- Cancel order if allowed.
- Store order and order items in PostgreSQL.

### 4.7 Payment Service

- Create payment request.
- Redirect/call external payment provider.
- Receive webhook from Momo/VNPay.
- Verify payment signature.
- Update payment status.
- Publish payment events.

### 4.8 Shipping Service

- Create shipping order with GHN/GHTK.
- Receive shipping status update.
- Update order delivery status.
- Support tracking number.

### 4.9 Notification Service

- Send email/SMS/push notification.
- Notify order created, payment success, shipment created, delivery updated.
- Use Kafka events as input.

### 4.10 Admin Service

- Approve/reject products.
- Verify seller.
- Handle reports.
- View system metrics.
- Manage categories and policies.

---

## 5. Non-Functional Requirements

| NFR | Requirement | Justification |
|---|---|---|
| Scalability | 5K RPS normal peak baseline, up to 50K RPS during flash sale | Flash sale traffic spike |
| Availability | 99.9% uptime, downtime < 8.76h/year | Marketplace cần luôn hoạt động |
| Latency | API P95 < 200ms, Search P95 < 500ms | Người dùng bỏ đi nếu chậm |
| Consistency | Payment/order strong consistency, catalog/search eventual consistency | CAP trade-off |
| Reliability | RPO < 1 phút, RTO < 5 phút cho hệ thống chính | Giảm mất dữ liệu |
| Durability | Order/payment data không được mất | Liên quan tiền và pháp lý |
| Security | JWT, input validation, payment signature verification | Bảo vệ user và transaction |
| Observability | Logs, metrics, tracing, alerting | Phát hiện lỗi nhanh |
| Maintainability | Microservices theo domain | Dễ mở rộng và deploy |
| Cost efficiency | Dùng cache/CDN để giảm DB load | Giảm chi phí hạ tầng |

---

## 6. Out-of-Scope

Các phần không đi sâu trong thiết kế này:

- Warehouse Management System chi tiết.
- AI recommendation engine.
- Payment processor internals.
- Live streaming shopping.
- Fraud detection bằng ML phức tạp.
- Dynamic pricing engine.
- Full accounting system.
- Deep seller finance settlement system.

---

## 7. High-Level Architecture

### 7.1 Architecture Layers

Hệ thống được chia thành 7 lớp:

1. **Client Layer**
   - Buyer Web: React.
   - Buyer Mobile: Flutter.
   - Seller Portal: React.
   - Admin Panel: React.

2. **Edge Layer**
   - CDN CloudFront.
   - Static assets caching.
   - Image/video delivery.
   - Optional edge security/WAF.

3. **Gateway Layer**
   - Load Balancer: Nginx hoặc AWS ALB.
   - API Gateway: Kong.
   - Responsibilities: routing, authentication, rate limiting, request validation, circuit breaker at gateway level.

4. **Application Service Layer**
   - User Service.
   - Product Catalog Service.
   - Search Service.
   - Cart Service.
   - Order Service.
   - Inventory Service.
   - Payment Service.
   - Shipping Service.
   - Notification Service.
   - Analytics Service.
   - Admin Service.

5. **Data Layer**
   - Redis Cluster: cache, cart, session, rate limit, inventory hot counters.
   - PostgreSQL Primary-Replica: users, orders, payments, sellers.
   - MongoDB Sharding: product catalog.
   - Elasticsearch: search index.
   - S3: product images, documents.
   - ClickHouse: analytics warehouse.

6. **Messaging Layer**
   - Kafka Event Bus.
   - Topics:
     - `product-created`
     - `product-updated`
     - `inventory-reserved`
     - `order-created`
     - `payment-success`
     - `payment-failed`
     - `shipment-created`
     - `notification-requested`

7. **Analytics and Observability Layer**
   - Kafka Connect.
   - Apache Flink for real-time stream processing.
   - Apache Spark for batch ETL.
   - ClickHouse for analytics.
   - Grafana for dashboard.
   - Prometheus/ELK/OpenTelemetry for metrics/logs/traces.

---

## 8. Component Design

### 8.1 User Service

**Responsibilities:**

- Register/login.
- JWT issue/refresh.
- User profile.
- Address management.

**Database:** PostgreSQL table `users`.

**Key APIs:**

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `GET /api/v1/users/me`
- `PUT /api/v1/users/me`

**Design notes:**

- Password hashed bằng bcrypt/argon2.
- JWT stored in HTTP-only cookie hoặc secure mobile storage.
- Session/token cache in Redis: `session:{token}`.

---

### 8.2 Product Catalog Service

**Responsibilities:**

- Manage product metadata.
- Manage variants, attributes, images.
- Manage seller product status.
- Publish product events to Kafka.

**Database:** MongoDB collection `products`.

**Storage:** S3 for images.

**Key APIs:**

- `POST /api/v1/seller/products`
- `PUT /api/v1/seller/products/{product_id}`
- `GET /api/v1/products/{product_id}`
- `DELETE /api/v1/seller/products/{product_id}`

**Design notes:**

- MongoDB phù hợp vì product schema linh hoạt theo category.
- Product detail cached in Redis: `product:{id}`.
- Images stored in S3, served through CDN.
- Product update publishes `product-updated` event để Search/Indexing service cập nhật Elasticsearch.

---

### 8.3 Search Service

**Responsibilities:**

- Index product documents.
- Search by keyword.
- Filter/sort products.
- Maintain Elasticsearch index.

**Database:** Elasticsearch index `products`.

**Key APIs:**

- `GET /api/v1/products/search`

**Design notes:**

- Elasticsearch dùng inverted index cho full-text search.
- Index fields:
  - `name`
  - `description`
  - `category`
  - `brand`
  - `price`
  - `rating`
  - `sold_count`
  - `seller_id`
  - `stock_status`
- Search results có thể cache ngắn hạn: `search:{keyword}:{filters}` TTL 300s.

---

### 8.4 Cart Service

**Responsibilities:**

- Add/remove/update cart items.
- Calculate subtotal.
- Check product status.
- Check inventory approximately.

**Database:** Redis.

**Key APIs:**

- `POST /api/v1/cart/items`
- `PUT /api/v1/cart/items/{product_id}`
- `DELETE /api/v1/cart/items/{product_id}`
- `GET /api/v1/cart`

**Design notes:**

- Redis key: `cart:{user_id}`.
- TTL: 1800s hoặc lâu hơn nếu user active.
- Cart không cần lưu PostgreSQL vì chưa phải transaction chính thức.
- Khi checkout, cart được convert thành order snapshot.

---

### 8.5 Inventory Service

**Responsibilities:**

- Reserve stock during checkout.
- Confirm stock after payment.
- Release stock if payment fails.
- Decrease stock after payment success.
- Prevent overselling.

**Database:** Redis for hot inventory counters, PostgreSQL for inventory ledger.

**Key APIs:**

- `POST /api/v1/inventory/reservations`
- `POST /api/v1/inventory/reservations/{reservation_id}/confirm`
- `POST /api/v1/inventory/reservations/{reservation_id}/release`

**Design notes:**

- Redis atomic decrement hoặc Lua script để tránh overselling.
- PostgreSQL lưu inventory ledger để audit.
- Inventory events published to Kafka.
- Catalog/search nhận event để cập nhật stock status eventual consistency.

---

### 8.6 Order Service

**Responsibilities:**

- Create order.
- Store order snapshot.
- Track order status.
- Publish order events.

**Database:** PostgreSQL tables `orders`, `order_items`.

**Key APIs:**

- `POST /api/v1/orders`
- `GET /api/v1/orders/{order_id}`
- `GET /api/v1/users/me/orders`
- `POST /api/v1/orders/{order_id}/cancel`

**Design notes:**

- Order phải lưu snapshot của product name, price, seller, quantity tại thời điểm mua.
- Không nên query lại catalog để hiển thị historical order.
- Order status:
  - `PENDING_PAYMENT`
  - `PAID`
  - `CONFIRMED`
  - `PACKING`
  - `SHIPPED`
  - `DELIVERED`
  - `CANCELLED`
  - `REFUNDED`

---

### 8.7 Payment Service

**Responsibilities:**

- Create payment session.
- Call Momo/VNPay.
- Verify webhook signature.
- Update payment status.
- Publish payment events.

**Database:** PostgreSQL table `payments`.

**Key APIs:**

- `POST /api/v1/payments`
- `POST /api/v1/payments/webhook/momo`
- `POST /api/v1/payments/webhook/vnpay`
- `GET /api/v1/payments/{payment_id}`

**Design notes:**

- Amount được lấy từ order trên server, không tin client.
- Payment là critical path, cần strong consistency.
- Webhook phải idempotent.
- Nếu payment success nhưng order update lỗi, retry mechanism và reconciliation job cần chạy.

---

### 8.8 Shipping Service

**Responsibilities:**

- Create shipment.
- Get shipping fee.
- Update tracking status.
- Receive webhook from GHN/GHTK.

**Database:** PostgreSQL table `shipments`.

**Key APIs:**

- `POST /api/v1/shipments`
- `GET /api/v1/shipments/{shipment_id}`
- `POST /api/v1/shipping/webhook/ghn`

**Design notes:**

- Shipping là external dependency, không nên block toàn bộ checkout.
- Nếu shipping provider timeout, order vẫn có thể ở trạng thái `PAID_PENDING_SHIPMENT`.
- Circuit breaker cần áp dụng cho shipping provider.

---

### 8.9 Notification Service

**Responsibilities:**

- Send email, SMS, push notification.
- Listen to Kafka events.
- Retry failed notification.

**Database:** PostgreSQL hoặc MongoDB cho notification history.

**Consumed events:**

- `order-created`
- `payment-success`
- `payment-failed`
- `shipment-created`
- `delivery-updated`

**Design notes:**

- Notification không nằm trong critical path.
- Nếu notification service down, Kafka giữ event để retry.

---

### 8.10 Analytics Service

**Responsibilities:**

- Collect user events, order events, seller events.
- Stream processing with Flink.
- Batch ETL with Spark.
- Store analytics in ClickHouse.
- Dashboard in Grafana.

**Data sources:**

- User events.
- Order events.
- Payment events.
- Product events.
- System logs.

**Use cases:**

- Real-time GMV dashboard.
- Flash sale monitoring.
- Seller performance.
- Product popularity.
- Error rate monitoring.

---

## 9. Data Design

### 9.1 PostgreSQL Tables

#### `users`

| Column | Type | Description |
|---|---|---|
| user_id | UUID PK | User ID |
| email | VARCHAR UNIQUE | Email |
| password_hash | TEXT | Hashed password |
| full_name | VARCHAR | Full name |
| phone | VARCHAR | Phone |
| role | ENUM | buyer/seller/admin |
| is_active | BOOLEAN | Account status |
| created_at | TIMESTAMP | Creation time |

#### `sellers`

| Column | Type | Description |
|---|---|---|
| seller_id | UUID PK | Seller ID |
| user_id | UUID FK | Owner user |
| business_name | VARCHAR | Business name |
| tax_id | VARCHAR | Tax ID |
| verified | BOOLEAN | Verification status |
| created_at | TIMESTAMP | Creation time |

#### `orders`

| Column | Type | Description |
|---|---|---|
| order_id | UUID PK | Order ID |
| user_id | UUID FK | Buyer ID |
| total_amount | DECIMAL | Total amount |
| status | ENUM | Order status |
| shipping_address | JSONB | Shipping address snapshot |
| payment_status | ENUM | Payment status |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Update time |

#### `order_items`

| Column | Type | Description |
|---|---|---|
| item_id | UUID PK | Item ID |
| order_id | UUID FK | Order ID |
| product_id | UUID | Product ID snapshot |
| seller_id | UUID | Seller ID snapshot |
| product_name | VARCHAR | Product name snapshot |
| quantity | INT | Quantity |
| unit_price | DECIMAL | Unit price |
| subtotal | DECIMAL | Subtotal |

#### `payments`

| Column | Type | Description |
|---|---|---|
| payment_id | UUID PK | Payment ID |
| order_id | UUID FK | Order ID |
| amount | DECIMAL | Amount |
| method | ENUM | Momo/VNPay/etc |
| transaction_ref | VARCHAR | External transaction ref |
| status | ENUM | PENDING/SUCCESS/FAILED/REFUNDED |
| paid_at | TIMESTAMP | Payment time |

#### `inventory_ledger`

| Column | Type | Description |
|---|---|---|
| ledger_id | UUID PK | Ledger ID |
| product_id | UUID | Product ID |
| seller_id | UUID | Seller ID |
| change_type | ENUM | RESERVE/CONFIRM/RELEASE/DECREASE |
| quantity | INT | Quantity change |
| order_id | UUID | Related order |
| created_at | TIMESTAMP | Creation time |

---

### 9.2 MongoDB Collections

#### `products`

```json
{
  "_id": "ObjectId",
  "seller_id": "UUID",
  "name": "Product name",
  "description": "Product description",
  "category": ["Electronics", "Phone"],
  "brand": "Apple",
  "attributes": {
    "color": "Black",
    "storage": "128GB"
  },
  "images": ["s3://bucket/image1.jpg"],
  "tags": ["iphone", "smartphone"],
  "variants": [
    {
      "variant_id": "UUID",
      "sku": "SKU-001",
      "price": 15000000,
      "stock": 100
    }
  ],
  "status": "active",
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

#### `seller_profiles`

```json
{
  "_id": "ObjectId",
  "seller_id": "UUID",
  "business_info": {},
  "documents": [],
  "bank_account": {},
  "rating": 4.8,
  "created_at": "timestamp"
}
```

---

### 9.3 Redis Keys

| Key | Type | TTL | Purpose |
|---|---|---:|---|
| `session:{token}` | STRING | 86400s | JWT/session validation |
| `cart:{user_id}` | HASH | 1800s | Shopping cart |
| `product:{product_id}` | JSON | 3600s + jitter | Product detail cache |
| `search:{query_hash}` | JSON | 300s | Search result cache |
| `inventory:{product_id}` | INT | 30s | Hot stock counter |
| `rate_limit:{ip_or_user}` | INT | 60s | Rate limiting |
| `idempotency:{request_id}` | STRING | 86400s | Idempotent payment/order requests |

---

## 10. API Design

### 10.1 Standard Response Format

```json
{
  "status": 200,
  "data": {},
  "message": "success",
  "timestamp": "2026-05-15T10:00:00Z"
}
```

### 10.2 Product APIs

```http
GET /api/v1/products/search
```

Input:

```json
{
  "keyword": "iphone",
  "category": "Electronics",
  "price_min": 1000000,
  "price_max": 20000000,
  "page": 1,
  "page_size": 20
}
```

Output:

```json
{
  "products": [],
  "total": 1000,
  "page": 1,
  "page_size": 20
}
```

```http
GET /api/v1/products/{product_id}
```

Output:

```json
{
  "product_id": "uuid",
  "seller_id": "uuid",
  "name": "iPhone 15",
  "price": 15000000,
  "images": [],
  "variants": [],
  "stock_status": "available"
}
```

---

### 10.3 Cart APIs

```http
POST /api/v1/cart/items
```

Input:

```json
{
  "product_id": "uuid",
  "variant_id": "uuid",
  "quantity": 2
}
```

Output:

```json
{
  "cart_id": "cart-001",
  "items": [],
  "subtotal": 30000000
}
```

```http
GET /api/v1/cart
```

Output:

```json
{
  "items": [],
  "subtotal": 30000000
}
```

---

### 10.4 Order APIs

```http
POST /api/v1/orders
```

Input:

```json
{
  "items": [
    {
      "product_id": "uuid",
      "variant_id": "uuid",
      "quantity": 2
    }
  ],
  "shipping_address": {
    "receiver_name": "Nguyen Van A",
    "phone": "0901234567",
    "address": "Ha Noi"
  },
  "payment_method": "momo"
}
```

Output:

```json
{
  "order_id": "uuid",
  "total_amount": 30000000,
  "status": "PENDING_PAYMENT",
  "payment_status": "PENDING"
}
```

```http
GET /api/v1/orders/{order_id}/status
```

Output:

```json
{
  "order_id": "uuid",
  "order_status": "PAID",
  "payment_status": "SUCCESS",
  "shipping_status": "PENDING"
}
```

---

### 10.5 Payment APIs

```http
POST /api/v1/payments
```

Input:

```json
{
  "order_id": "uuid",
  "method": "momo"
}
```

Output:

```json
{
  "payment_id": "uuid",
  "transaction_id": "external-transaction-id",
  "checkout_url": "https://payment-provider.com/..."
}
```

Important rule:

> Amount must be calculated from server-side order, not from client input.

---

### 10.6 Seller APIs

```http
POST /api/v1/seller/products
PUT /api/v1/seller/products/{product_id}
DELETE /api/v1/seller/products/{product_id}
GET /api/v1/seller/products
GET /api/v1/seller/orders
GET /api/v1/seller/dashboard
```

---

### 10.7 Admin APIs

```http
POST /api/v1/admin/products/{product_id}/approve
POST /api/v1/admin/products/{product_id}/reject
POST /api/v1/admin/sellers/{seller_id}/verify
GET /api/v1/admin/reports
GET /api/v1/admin/system-metrics
```

---

## 11. Core Request Flows

### 11.1 Normal Flow — Buyer Places an Order

**Scenario:** Buyer tìm sản phẩm, thêm vào giỏ, checkout, thanh toán bằng Momo.

Flow:

```text
Buyer
  → CDN
  → Load Balancer
  → API Gateway/Kong
  → Product Catalog/Search Service
  → Redis cache hit
  → Buyer views product detail
  → Cart Service
  → Redis cart
  → Order Service
  → Inventory Service
  → Redis atomic stock reservation
  → PostgreSQL insert order
  → Kafka publish order-created
  → Payment Service
  → Momo
  → Momo webhook
  → Payment Service verifies signature
  → PostgreSQL update payment status
  → PostgreSQL update order status
  → Kafka publish payment-success
  → Notification Service sends notification
  → Shipping Service creates shipment
  → Buyer tracks order
```

Detailed steps:

1. Buyer search product.
2. Search Service returns result from Elasticsearch.
3. Buyer opens product detail.
4. Product Catalog Service checks Redis.
   - Cache hit: return data.
   - Cache miss: read MongoDB, then set Redis cache.
5. Buyer adds item to cart.
6. Cart Service stores cart in Redis.
7. Buyer checkout.
8. Order Service validates cart and calculates total on server.
9. Inventory Service reserves stock in Redis using atomic operation.
10. Order Service inserts order into PostgreSQL with status `PENDING_PAYMENT`.
11. Order Service publishes `order-created`.
12. Payment Service creates payment with Momo/VNPay.
13. Buyer completes payment.
14. Payment provider sends webhook.
15. Payment Service verifies signature and idempotency.
16. Payment Service updates payment and order in PostgreSQL transaction.
17. Payment Service publishes `payment-success`.
18. Notification Service sends confirmation.
19. Shipping Service creates shipment.
20. Buyer receives tracking status.

---

### 11.2 Peak Traffic Flow — Flash Sale

**Scenario:** 200,000 concurrent users join a flash sale, 50K RPS peak.

Main risks:

- Search overload.
- Product detail overload.
- Cart overload.
- Inventory overselling.
- Order service overload.
- Payment provider timeout.
- Database connection exhaustion.

Flash sale design:

```text
Buyer
  → CDN
  → WAF / Rate Limit
  → Load Balancer
  → API Gateway
  → Queue/Waiting Room optional
  → Search/Product Service
  → Redis Cache
  → Inventory Reservation
  → Order Service
  → Kafka
  → Payment Service
  → Payment Provider
```

Optimizations:

1. **CDN cache**
   - Product images cached at edge.
   - Static JS/CSS cached.
   - Flash sale landing page cached for 1–5 minutes.

2. **Redis preloading**
   - Flash sale product details preloaded into Redis before event.
   - Inventory hot counters loaded into Redis.

3. **Search optimization**
   - Flash sale query cached.
   - Popular search keywords precomputed.
   - Elasticsearch replicas scaled before event.

4. **Inventory reservation**
   - Redis Lua script atomic decrement.
   - If stock <= 0, return `OUT_OF_STOCK` immediately.
   - No direct PostgreSQL write for every failed reservation.

5. **Order creation**
   - Order Service stateless, horizontally scaled.
   - Kafka absorbs burst.
   - PostgreSQL connection pool controlled by PgBouncer/HAProxy.

6. **Payment**
   - Payment is async after order creation.
   - Payment provider timeout handled by circuit breaker.
   - Idempotency key prevents duplicate payment creation.

7. **Backpressure**
   - API Gateway rate limits abusive clients.
   - Waiting room/queue page for extreme traffic.
   - Non-critical features disabled temporarily:
     - Reviews.
     - Recommendations.
     - Analytics heavy queries.
     - Seller dashboard refresh.

8. **Observability**
   - Real-time dashboard:
     - RPS.
     - Error rate.
     - P95 latency.
     - Redis hit rate.
     - Kafka lag.
     - DB connection usage.
     - Payment success rate.

---

### 11.3 Failure Case — Payment Provider Timeout

**Scenario:** Momo/VNPay timeout during payment.

Failure flow:

```text
Buyer
  → Payment Service
  → Momo/VNPay
  → Timeout after 5 seconds
  → Circuit Breaker opens
  → Order remains PENDING_PAYMENT
  → Retry later or user retries
  → Inventory reservation expires if not paid
```

Handling:

1. Payment Service sets timeout, for example 5s.
2. If timeout occurs, do not mark order failed immediately.
3. Order remains `PENDING_PAYMENT`.
4. Inventory reservation has TTL, for example 10 minutes.
5. If payment success webhook arrives later, system processes it idempotently.
6. If payment never succeeds, reservation expires.
7. Expired reservation event releases stock.
8. Circuit breaker opens after 5 consecutive failures.
9. During OPEN state, new payment calls to that provider are rejected fast.
10. After 30s, circuit breaker enters HALF-OPEN.
11. One test call is allowed.
12. If success, circuit closes.
13. If fail, circuit opens again.

Circuit breaker states:

```text
CLOSED → 5 failures → OPEN → wait 30s → HALF-OPEN → 1 test call
       → success → CLOSED
       → failure → OPEN
```

Why this matters:

- Prevents one external provider failure from blocking the whole system.
- Avoids thread/connection exhaustion.
- Gives fast feedback to users.

---

## 12. Cache Design

### 12.1 Why Cache?

Redis nhanh hơn PostgreSQL rất nhiều:

| Metric | Redis | PostgreSQL |
|---|---:|---:|
| Latency | ~0.1ms | ~10ms |
| Throughput | ~100K ops/sec | ~5K queries/sec |

Với peak RPS khoảng 4,630–50,000, PostgreSQL không thể tự xử lý toàn bộ request đọc nếu không có cache.

### 12.2 Cache Strategy

Selected strategy:

> **Cache Aside + Invalidation**

Read path:

```text
App receives request
  → Read Redis
  → Hit: return data
  → Miss: read database
  → Write to Redis with TTL
  → Return data
```

Write path:

```text
App updates database
  → Delete cache key
  → Next read will miss and reload from DB
```

Why delete cache instead of setting new value directly?

- Tránh race condition khi có nhiều request update đồng thời.
- Tránh stale cache.
- Đơn giản và reliable hơn trong distributed system.

### 12.3 Cache TTL

| Data | TTL |
|---|---:|
| Product detail | 3600s + jitter |
| Cart | 1800s |
| Search result | 300s |
| Session/token | 86400s |
| Inventory count | 30s |
| Rate limit | 60s |
| Flash sale product | 60–300s |

### 12.4 Thundering Herd Problem

Problem:

- Nhiều cache key hết hạn cùng lúc.
- Hàng nghìn request cùng query database.
- Database bị overload.

Fix:

- TTL jitter:

```text
actual_ttl = base_ttl + random(0, 300)
```

Additional protections:

- Singleflight/request coalescing.
- Preload hot keys before flash sale.
- Use Redis cluster.
- Graceful fallback to stale cache if DB is overloaded.

---

## 13. Data Consistency Design

### 13.1 Strong Consistency Areas

Use PostgreSQL transaction for:

- Order creation.
- Payment status update.
- User account changes.
- Seller verification.
- Financial ledger.

Example:

```text
BEGIN;
  UPDATE payments SET status = 'SUCCESS';
  UPDATE orders SET status = 'PAID';
  INSERT inventory_ledger ...
COMMIT;
```

### 13.2 Eventual Consistency Areas

Use Kafka events for:

- Product search index update.
- Product detail cache invalidation.
- Inventory status sync.
- Notification.
- Analytics.
- Seller dashboard aggregation.

Example:

```text
Product updated in MongoDB
  → publish product-updated
  → Search Service updates Elasticsearch
  → Redis cache key deleted
```

### 13.3 Idempotency

Important for:

- Payment webhook.
- Shipping webhook.
- Kafka consumer retry.
- Order creation.

Implementation:

- Use `idempotency_key`.
- Store processed request IDs in Redis/PostgreSQL.
- If same request arrives again, return previous result.

---

## 14. Database Scaling Design

### 14.1 PostgreSQL

Used for:

- Users.
- Orders.
- Payments.
- Sellers.
- Inventory ledger.

Scaling:

- Primary-Replica architecture.
- Writes go to Primary.
- Reads go to Replicas.
- HAProxy/PgBouncer manages connections.
- Payment uses stronger replication consistency.
- Catalog-related reads can tolerate async replica.

Failover:

```text
Primary fails
  → HAProxy detects failure
  → Replica promoted
  → App reconnects to new primary
  → RTO < 5 minutes
```

Backup:

- Daily full backup to S3.
- Hourly incremental backup.
- Point-in-time recovery if possible.

---

### 14.2 MongoDB

Used for:

- Product catalog.
- Seller profiles.

Scaling:

- Sharding by `seller_id` or hash-based shard key.
- 3 shards for initial scale.
- `mongos` routes queries.
- Product catalog schema linh hoạt theo category.

Shard example:

| Shard | Data |
|---|---|
| Shard 1 | seller_id range/hash group 1 |
| Shard 2 | seller_id range/hash group 2 |
| Shard 3 | seller_id range/hash group 3 |

Risk:

- Hot shard nếu một seller quá lớn.

Mitigation:

- Use hashed shard key.
- Split large seller into sub-keys.
- Monitor shard size and query distribution.

---

### 14.3 Elasticsearch

Used for:

- Product search.
- Filtering.
- Sorting.

Scaling:

- Multiple primary shards.
- Replica shards for read throughput and failover.
- Index lifecycle management.
- Reindex when mapping changes.

---

### 14.4 Redis

Used for:

- Cache.
- Cart.
- Session.
- Rate limit.
- Inventory hot counters.

Scaling:

- Redis Cluster.
- Primary-replica per shard.
- Persistence optional depending on data type.
- Critical counters also mirrored to PostgreSQL ledger.

---

### 14.5 S3

Used for:

- Product images.
- Seller documents.
- Backup files.

Benefits:

- Durable.
- Cheap.
- Integrates with CDN.
- Good for static/object storage.

---

### 14.6 ClickHouse

Used for:

- Analytics.
- Real-time dashboard.
- GMV tracking.
- Event aggregation.

Why not use PostgreSQL for analytics?

- PostgreSQL tốt cho transaction.
- ClickHouse tốt cho columnar aggregation trên lượng dữ liệu lớn.

---

## 15. Messaging Design

### 15.1 Why Kafka?

Kafka được chọn vì:

- High throughput.
- Durable event log.
- Decouples services.
- Supports retry and replay.
- Good for event-driven architecture.

### 15.2 Main Topics

| Topic | Producer | Consumer |
|---|---|---|
| `product-created` | Product Catalog | Search, Analytics |
| `product-updated` | Product Catalog | Search, Cache Invalidation |
| `inventory-reserved` | Inventory | Order, Analytics |
| `order-created` | Order | Notification, Analytics, Inventory |
| `payment-success` | Payment | Order, Shipping, Notification |
| `payment-failed` | Payment | Order, Inventory, Notification |
| `shipment-created` | Shipping | Notification, Analytics |
| `delivery-updated` | Shipping | Order, Notification |

### 15.3 Consumer Reliability

- Consumer groups for parallel processing.
- Offset commit after successful processing.
- Dead Letter Queue for failed events.
- Retry with exponential backoff.
- Idempotent consumers.

---

## 16. Scalability Strategy

### 16.1 Application Scaling

Application services are stateless.

Scaling method:

```text
More requests
  → Load Balancer distributes traffic
  → Auto Scaling Group adds instances
  → Health check removes unhealthy instances
```

Auto-scaling policy:

| Metric | Scale Out | Scale In |
|---|---|---|
| CPU | > 70% for 5 min | < 30% for 10 min |
| RPS | High sustained RPS | Low sustained RPS |
| Latency | P95 > 200ms | P95 stable low |

Deployment:

- Multiple availability zones.
- Rolling deployment.
- Blue-green or canary for critical services.

---

### 16.2 Database Scaling

PostgreSQL:

- Read replicas.
- Connection pooling.
- Index optimization.
- Partitioning for large tables like orders.

MongoDB:

- Sharding.
- Indexes on category, price, seller_id.
- Avoid unbounded queries.

Elasticsearch:

- More replicas for read-heavy search.
- Index rollover for time-based logs/events.

Redis:

- Cluster mode.
- Hot keys monitoring.
- Preload flash sale keys.

---

### 16.3 CDN and Performance Stack

Performance target:

```text
CDN cache hit: ~20ms
Redis: ~0.1ms
PostgreSQL indexed query: ~10ms
Search: P95 < 500ms
API: P95 < 200ms
```

CDN cache policy:

| Asset | TTL |
|---|---:|
| Images | 7 days |
| JS/CSS | 30 days |
| HTML | 5 minutes |
| API read-only | 1 minute |
| Cart | Not cached |
| Orders | Not cached |
| Payments | Not cached |

Target:

- 90% static/read requests served from cache.
- Critical transaction APIs bypass cache.

---

## 17. Reliability and Fault Tolerance

### 17.1 Failure Scenarios

| Component | Failure | Response | RTO/RPO |
|---|---|---|---|
| App server | Instance dies | Load balancer removes, ASG replaces | ~2 min |
| Redis | Cache node dies | Fallback to DB, replica promotion | Instant to <1 min |
| PostgreSQL primary | Primary dies | Replica promoted | RTO < 5 min |
| PostgreSQL replica | Replica dies | Remove from pool, rebuild | No data loss |
| MongoDB shard | Shard fails | Replica set failover | <1–2 min |
| Kafka node | Broker dies | Leader election | <1 min |
| Momo/VNPay | Timeout | Circuit breaker opens | Instant |
| GHN/GHTK | Timeout | Retry + circuit breaker | Instant |
| CDN | Edge issue | Fallback to origin/S3 | Depends provider |

### 17.2 Circuit Breaker

Used for:

- Payment providers.
- Shipping providers.
- External notification providers.
- Internal slow services.

State machine:

```text
CLOSED
  → normal operation
  → after 5 failures
OPEN
  → fail fast for 30s
  → no real calls
HALF-OPEN
  → allow 1 test call
  → success → CLOSED
  → failure → OPEN
```

### 17.3 Retry Policy

Retry only safe operations:

- GET product detail.
- Search.
- Notification send.
- Shipping status fetch.

Careful or no retry:

- Payment creation.
- Order creation.
- Stock decrease.

For unsafe operations:

- Use idempotency key.
- Use exponential backoff.
- Use jitter.
- Use dead letter queue.

---

## 18. Security Design

### 18.1 Authentication and Authorization

- JWT for API authentication.
- HTTP-only cookie preferred for web.
- Refresh token rotation.
- Role-based access control:
  - Buyer.
  - Seller.
  - Admin.

### 18.2 API Security

- Rate limiting by IP/user.
- Input validation.
- Request size limit.
- SQL injection prevention.
- NoSQL injection prevention.
- CORS policy.
- HTTPS everywhere.

### 18.3 Payment Security

- Verify Momo/VNPay webhook signature.
- Do not trust client amount.
- Use idempotency key.
- Store external transaction reference.
- Reconciliation job daily.

### 18.4 Data Security

- Password hashing with bcrypt/argon2.
- Encrypt sensitive data at rest.
- Limit admin access.
- Audit logs for admin actions.
- Backup encryption.

---

## 19. Capacity Estimation

### 19.1 Traffic Assumptions

| Metric | Value |
|---|---:|
| MAU | 10,000,000 |
| DAU | 2,000,000 |
| Requests/user/day | 20 |
| Total requests/day | 40,000,000 |
| Normal RPS | ~463 RPS |
| Write ratio | 20% |
| Normal write RPS | ~93 RPS |
| Peak multiplier | 10x |
| Peak RPS | ~4,630 RPS |
| Flash sale target | up to 50,000 RPS |
| Peak concurrent users | ~200,000 |

### 19.2 Storage Estimation

| Data | Calculation | Result |
|---|---|---:|
| Products | 100M × 1KB | ~100GB |
| Orders/day | 500K × 1KB | ~500MB/day |
| Images | 100M × 3 × 200KB | ~60TB |
| Logs/day | 2M × 100 events × 100B | ~20GB/day |
| Total storage growth | Approx. | ~500GB/day, ~180TB/year |

### 19.3 Read/Write Ratio

Estimated:

```text
Read : Write = 80 : 20
```

Implication:

- Cache and CDN are critical.
- Read replicas help.
- Search must be optimized.
- Writes must be controlled for order/payment/inventory.

---

## 20. Technology Justification

### 20.1 Why Microservices?

Advantages:

- Each domain can scale independently.
- Easier deployment.
- Fault isolation.
- Teams can work independently.
- Different databases can be used per service.

Trade-off:

- More operational complexity.
- Need service discovery, monitoring, tracing.
- Distributed transactions are harder.

Decision:

> Use microservices because marketplace has clearly separated domains: user, catalog, cart, order, payment, inventory, shipping, notification.

---

### 20.2 Why React and Flutter?

| Need | Technology | Reason |
|---|---|---|
| Web buyer/seller/admin | React | Popular, fast UI development, large ecosystem |
| Mobile buyer app | Flutter | Cross-platform iOS/Android, good performance |

Trade-off:

- Flutter needs native bridge for some features.
- React web needs SEO optimization if public pages matter.

---

### 20.3 Why CloudFront/CDN?

Without CDN:

```text
Vietnam user → Singapore origin → ~500ms
```

With CDN:

```text
Vietnam user → Hanoi edge → ~20ms
```

Benefits:

- Faster static delivery.
- Lower origin load.
- Better flash sale resilience.
- Cheaper bandwidth from origin.

Trade-off:

- Cache invalidation complexity.
- Need correct TTL policy.

---

### 20.4 Why Nginx/AWS ALB?

Responsibilities:

- Load balancing.
- Health check.
- TLS termination.
- Route to app instances.

Benefits:

- Mature.
- Reliable.
- Easy horizontal scaling.

---

### 20.5 Why Kong API Gateway?

Responsibilities:

- Routing.
- Authentication.
- Rate limiting.
- Logging.
- Plugin ecosystem.
- Centralized API policy.

Benefits:

- Reduces duplicated logic in services.
- Good for marketplace with many APIs.

Trade-off:

- Gateway can become bottleneck.
- Need horizontal scaling and monitoring.

---

### 20.6 Why Redis?

Used for:

- Cache.
- Cart.
- Session.
- Rate limit.
- Inventory hot counters.

Benefits:

- Very low latency.
- TTL support.
- Atomic operations.
- Lua scripting.
- Cluster mode.

Trade-off:

- RAM cost.
- Need cache invalidation strategy.
- Not suitable as source of truth for money/order data.

---

### 20.7 Why PostgreSQL?

Used for:

- Users.
- Orders.
- Payments.
- Sellers.
- Inventory ledger.

Benefits:

- ACID transactions.
- Strong consistency.
- Mature indexing.
- Reliable replication.
- Good for financial data.

Trade-off:

- Harder horizontal write scaling.
- Need replicas, pooling, partitioning.

---

### 20.8 Why MongoDB?

Used for:

- Product catalog.
- Seller profiles.

Benefits:

- Flexible schema.
- Good for different product categories.
- Horizontal scaling with sharding.
- JSON-like document model.

Trade-off:

- Complex transactions across documents.
- Need careful indexing and shard key design.

---

### 20.9 Why Elasticsearch?

Used for:

- Product search.
- Filtering.
- Sorting.

Benefits:

- Inverted index.
- Full-text search.
- Faceted search.
- Fast query over large product catalog.

Trade-off:

- Additional infrastructure.
- Index consistency is eventual.
- Needs reindex strategy.

---

### 20.10 Why Kafka?

Used for:

- Event bus.
- Async communication.
- Decoupling services.
- Retry/replay.

Benefits:

- High throughput.
- Durable logs.
- Good for order/payment/inventory/notification events.

Trade-off:

- Operational complexity.
- Consumer lag monitoring required.

---

### 20.11 Why Flink, Spark, ClickHouse?

| Need | Technology | Reason |
|---|---|---|
| Real-time stream processing | Flink | Low-latency event processing |
| Batch ETL | Spark | Mature batch processing |
| Analytics warehouse | ClickHouse | Fast columnar aggregation |

Why not PostgreSQL for analytics?

- PostgreSQL is optimized for OLTP.
- ClickHouse is optimized for OLAP and large aggregation.

---

## 21. Cost vs Performance Considerations

### 21.1 Cost Drivers

Main cost drivers:

1. Compute instances.
2. Database storage.
3. Redis memory.
4. Elasticsearch storage.
5. S3 object storage.
6. CDN bandwidth.
7. Kafka cluster.
8. Monitoring/logging storage.

### 21.2 Cost Optimization

| Area | Optimization |
|---|---|
| CDN | Cache static assets to reduce origin traffic |
| Redis | Cache hot data to reduce DB queries |
| PostgreSQL | Read replicas only for heavy read traffic |
| MongoDB | Shard only when data exceeds single-node capacity |
| Elasticsearch | Use replicas based on search QPS |
| Kafka | Size by throughput and retention requirement |
| S3 | Use lifecycle policy for old objects/logs |
| Logs | Aggregate and rotate logs |
| App instances | Auto-scale based on CPU/RPS/latency |

### 21.3 Performance vs Cost Trade-off

| Decision | Performance Benefit | Cost Impact |
|---|---|---|
| More Redis cache | Lower DB load, lower latency | Higher RAM cost |
| More PostgreSQL replicas | Better read scalability | Higher DB cost |
| More Elasticsearch replicas | Better search throughput | Higher CPU/storage |
| CDN caching | Faster delivery, lower origin load | CDN cost, cache management |
| Kafka buffering | Handles burst traffic | Kafka ops/storage cost |
| Auto-scaling | Handles variable traffic | Pay more during peak |

Decision principle:

> Spend more on components that protect correctness and user-critical paths: payment, order, inventory, authentication. Optimize cost on non-critical paths: analytics retention, logs, recommendations.

---

## 22. Final Architecture Refinement

Compared with the initial architecture, the final design adds/refines:

1. **Inventory Service separated from Order Service**
   - Better control for stock reservation.
   - Prevents overselling.
   - Supports flash sale.

2. **Shipping Service separated**
   - External shipping provider should not block payment/order core.
   - Circuit breaker can isolate provider failure.

3. **Idempotency design**
   - Critical for payment webhook, order creation, Kafka retry.

4. **Cache invalidation design**
   - Delete cache on write.
   - TTL jitter to avoid thundering herd.

5. **Event-driven notification/analytics**
   - Notification and analytics do not block checkout.

6. **Flash sale readiness**
   - Redis preloading.
   - Rate limiting.
   - Waiting room optional.
   - Disable non-critical features.

7. **Strong vs eventual consistency boundary**
   - Strong for payment/order.
   - Eventual for catalog/search/notification/analytics.

8. **Observability**
   - Metrics, logs, tracing, alerting are part of the architecture, not afterthought.

---

## 23. End-to-End Scenarios for Week 5

### Scenario 1: Normal Flow

**Title:** Buyer buys a product successfully.

**Actors:** Buyer, Product Catalog, Cart, Order, Payment, Notification, Shipping.

**Steps:**

1. Buyer searches product.
2. Search Service returns results from Elasticsearch.
3. Buyer opens product detail.
4. Product Catalog returns data from Redis cache.
5. Buyer adds product to cart.
6. Cart Service stores cart in Redis.
7. Buyer checks out.
8. Order Service creates order in PostgreSQL.
9. Inventory Service reserves stock in Redis.
10. Payment Service redirects buyer to Momo.
11. Buyer pays successfully.
12. Momo sends webhook.
13. Payment Service verifies webhook.
14. Payment and order status become success.
15. Shipping Service creates shipment.
16. Notification Service sends confirmation.

**Success criteria:**

- Order status is `PAID`.
- Payment status is `SUCCESS`.
- Inventory is decreased.
- Buyer receives notification.
- Shipment is created.

---

### Scenario 2: Peak Traffic Flow

**Title:** Flash sale with 50K RPS.

**Actors:** Buyers, CDN, API Gateway, Redis, Inventory, Order, Kafka, Payment.

**Steps:**

1. Flash sale starts.
2. CDN serves static assets and landing page.
3. API Gateway applies rate limiting.
4. Product details are served from Redis.
5. Inventory counters are in Redis.
6. Buyers reserve stock using atomic Redis operation.
7. Failed reservations return `OUT_OF_STOCK` immediately.
8. Successful reservations create orders.
9. Kafka absorbs order/payment events.
10. Payment Service handles provider calls asynchronously.
11. Monitoring dashboard shows RPS, latency, error rate, Kafka lag.
12. Auto-scaling adds app instances if CPU/latency exceeds threshold.

**Success criteria:**

- System handles 50K RPS target.
- No database overload.
- No overselling.
- Error rate remains acceptable.
- Payment failures are isolated.

---

### Scenario 3: Failure Case

**Title:** Momo payment provider timeout.

**Actors:** Buyer, Payment Service, Momo, Order Service, Inventory Service, Circuit Breaker.

**Steps:**

1. Buyer creates order.
2. Inventory is reserved.
3. Payment Service calls Momo.
4. Momo times out.
5. Circuit breaker records failure.
6. After 5 failures, circuit opens.
7. New payment attempts fail fast or show temporary provider issue.
8. Existing order remains `PENDING_PAYMENT`.
9. Inventory reservation expires after TTL.
10. Expired reservation releases stock.
11. If Momo webhook arrives later, Payment Service processes it idempotently.
12. After 30s, circuit enters HALF-OPEN and tests one request.
13. If successful, circuit closes.

**Success criteria:**

- No duplicate payment.
- No wrong order status.
- Stock is eventually released.
- Payment provider failure does not crash the whole system.
- User receives clear status.

---

## 24. Reflection

### 24.1 What We Did Well

1. **Clear separation of concerns**
   - User, catalog, cart, order, payment, inventory, shipping are separated.

2. **Correct consistency model**
   - Payment/order use strong consistency.
   - Search/catalog/notification/analytics use eventual consistency.

3. **Scalability-focused design**
   - Stateless services.
   - Load balancing.
   - Auto-scaling.
   - Redis cache.
   - CDN.
   - Kafka buffering.

4. **Reliability-focused design**
   - PostgreSQL replication.
   - Circuit breaker.
   - Retry with idempotency.
   - Dead letter queue.
   - Monitoring.

5. **Marketplace-specific design**
   - Multi-seller.
   - Product catalog flexibility.
   - Inventory reservation.
   - Flash sale handling.

---

### 24.2 Limitations

1. **Operational complexity**
   - Many services and databases require strong DevOps capability.

2. **Distributed transaction complexity**
   - Order, payment, inventory need careful idempotency and reconciliation.

3. **Cache consistency risk**
   - Cache invalidation can still cause temporary stale data.

4. **Hot shard risk**
   - MongoDB sharding can have hot shards if seller distribution is skewed.

5. **External dependency risk**
   - Momo/VNPay/GHN/GHTK can be unavailable.

6. **Cost**
   - Kafka, Elasticsearch, Redis, ClickHouse, monitoring stack increase infrastructure cost.

7. **Fraud detection not fully covered**
   - Basic rules can be added, but ML fraud detection is out of scope.

---

### 24.3 What to Improve

1. **Add real deployment blueprint**
   - Kubernetes manifests.
   - Helm charts.
   - CI/CD pipeline.

2. **Add detailed disaster recovery plan**
   - Multi-region backup.
   - RPO/RTO testing.
   - Failover drill.

3. **Improve fraud detection**
   - Rule-based first.
   - ML-based later.

4. **Improve seller settlement**
   - Wallet, commission, refund, payout.

5. **Improve search relevance**
   - Ranking by sales, rating, ads, personalization.

6. **Improve flash sale queue**
   - Token-based queue.
   - Virtual waiting room.
   - Lottery-style reservation for extreme traffic.

7. **Improve observability**
   - Distributed tracing.
   - SLO/SLI dashboard.
   - Alerting per service.

---

## 25. Presentation Storyline for Week 5

### Slide 1: Title

**E-Commerce Marketplace Platform — Final System Design**

Content:

- Group 1.
- System Engineering Project.
- Shopee-like marketplace for Vietnam.
- Focus: scalability, reliability, consistency, cost-performance.

---

### Slide 2: What We Deliver in Week 5

Content:

- Final architecture refinement.
- Three end-to-end scenarios:
  - Normal flow.
  - Flash sale peak traffic.
  - Failure case.
- Technology justification.
- Reflection and limitations.

---

### Slide 3: Final Architecture Overview

Show 7-layer architecture:

```text
Client → CDN → Load Balancer/API Gateway → Services → Data → Kafka → Analytics
```

Key message:

> We separate critical transaction path from non-critical async path.

---

### Slide 4: Normal Flow

Content:

- Search → Product detail → Cart → Order → Inventory reservation → Payment → Shipping → Notification.

Key message:

> Order/payment use PostgreSQL strong consistency; notification/shipping are async.

---

### Slide 5: Flash Sale Flow

Content:

- CDN cache.
- Redis preload.
- Atomic inventory reservation.
- Kafka buffering.
- Auto-scaling.
- Rate limiting.
- Non-critical features disabled.

Key message:

> Flash sale is handled by protecting database and controlling traffic before critical services.

---

### Slide 6: Failure Case

Content:

- Payment provider timeout.
- Circuit breaker.
- Idempotent webhook.
- Reservation TTL.
- Stock release.

Key message:

> One external provider failure must not collapse the whole marketplace.

---

### Slide 7: Technology Justification

Table:

| Need | Technology | Why |
|---|---|---|
| Web UI | React | Fast frontend development |
| Mobile | Flutter | Cross-platform |
| Cache | Redis | Low latency, TTL, atomic ops |
| Orders/Payments | PostgreSQL | ACID |
| Catalog | MongoDB | Flexible schema |
| Search | Elasticsearch | Full-text search |
| Messaging | Kafka | High throughput, durable events |
| Analytics | ClickHouse | Fast aggregation |
| CDN | CloudFront | Low latency static delivery |

---

### Slide 8: Cost vs Performance

Content:

- Redis reduces DB load.
- CDN reduces origin traffic.
- Kafka absorbs burst.
- Auto-scaling matches cost to demand.
- ClickHouse avoids overloading transaction DB with analytics.

Key message:

> We spend cost where it protects correctness and user experience.

---

### Slide 9: Reflection

Content:

- Strengths:
  - Scalable architecture.
  - Clear consistency boundaries.
  - Reliability patterns.
  - Marketplace-specific inventory design.

- Limitations:
  - Operational complexity.
  - External provider dependency.
  - Cache consistency risk.
  - Fraud detection not fully covered.

---

### Slide 10: What to Improve

Content:

- Kubernetes deployment.
- CI/CD.
- Multi-region DR.
- Advanced fraud detection.
- Seller settlement.
- Search ranking.
- Flash sale virtual waiting room.

---

## 26. Tutor Q&A Preparation

### Q1: Why not use one database for everything?

**Answer:**

We do not use one database because each domain has different requirements.

- Payment and order need ACID transactions, so PostgreSQL is suitable.
- Product catalog needs flexible schema, so MongoDB is suitable.
- Search needs inverted index, so Elasticsearch is suitable.
- Cache and cart need very low latency, so Redis is suitable.
- Analytics needs columnar aggregation, so ClickHouse is suitable.

This is called **polyglot persistence**.

---

### Q2: How do you prevent overselling during flash sale?

**Answer:**

We use Redis hot inventory counters with atomic decrement or Lua script.

Flow:

```text
Checkout request
  → Redis atomic decrement
  → if stock > 0, reservation succeeds
  → if stock <= 0, return OUT_OF_STOCK immediately
```

Successful reservation creates an order. If payment fails or expires, reservation is released. PostgreSQL inventory ledger is used for audit and source of truth.

---

### Q3: What happens if payment webhook arrives twice?

**Answer:**

Payment webhook processing is idempotent.

We store payment status and external transaction reference. If the same webhook arrives again:

- Check idempotency key or transaction reference.
- If already processed, return previous result.
- Do not update order/payment twice.

---

### Q4: What happens if Momo is down?

**Answer:**

Payment Service uses circuit breaker.

After several consecutive failures:

```text
CLOSED → OPEN → HALF-OPEN → CLOSED
```

During OPEN state, system fails fast and avoids exhausting threads/connections. Existing orders remain `PENDING_PAYMENT`, and inventory reservation expires after TTL if payment never succeeds.

---

### Q5: Why delete cache instead of updating cache after DB write?

**Answer:**

Deleting cache avoids race conditions.

If two requests update the same product at the same time, directly writing cache can store stale data. By deleting cache, the next read will reload from the database and produce correct data.

---

### Q6: What is thundering herd and how do you fix it?

**Answer:**

Thundering herd happens when many cache keys expire at the same time, causing thousands of requests to hit the database together.

Fix:

- TTL jitter.
- Singleflight/request coalescing.
- Preload hot keys before flash sale.
- Graceful fallback to stale cache when database is overloaded.

---

### Q7: What is your biggest limitation?

**Answer:**

The biggest limitation is operational complexity.

The system uses many components: Redis, PostgreSQL, MongoDB, Elasticsearch, Kafka, ClickHouse, CDN, external payment and shipping providers. This improves scalability and reliability, but requires strong DevOps, monitoring, backup, and incident response processes.

---

## 27. Final Conclusion

The final design is a scalable and reliable marketplace system for the Vietnam market. It uses microservices to separate business domains, polyglot persistence to match each data requirement, Redis and CDN to reduce latency and database load, Kafka to decouple services, and PostgreSQL to protect order/payment consistency.

The key design principle is:

> Keep the critical path safe and consistent, while moving non-critical work to async event-driven processing.

This allows the system to support normal marketplace traffic and handle flash sale peak traffic without collapsing the database or external services.