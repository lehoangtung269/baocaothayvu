# Final Report — E-Commerce Marketplace Platform

**Project:** System Engineering Project  
**System:** E-Commerce Marketplace Platform — Shopee-like Marketplace for Vietnam Market  
**Model:** Multi-seller B2C marketplace  
**Prepared for:** System Engineering course  
**Focus:** Reliability, scalability, maintainability, end-to-end data flow, distributed system design

---

## Image Placeholder Checklist

> **NOTE:** Đây là các vị trí cần chèn hình vào báo cáo. Các dòng placeholder được để trống và ghi chú rõ để bạn biết nên chèn hình nào.

| Hình | Vị trí trong báo cáo | Nội dung hình cần chèn |
|---|---|---|
| Hình 1 | Section 3.1 Architecture Diagram | Tổng quan kiến trúc 7 lớp: Client → CDN → Load Balancer/API Gateway → Services → Data Layer → Kafka → Analytics |
| Hình 2 | Section 3.2.1 Request Flow | Flow người dùng tìm sản phẩm, thêm vào giỏ, checkout, thanh toán, nhận đơn hàng |
| Hình 3 | Section 3.2.2 Data Pipeline | Data pipeline: ingestion → Kafka/Flink/Spark → storage → dashboard/report |
| Hình 4 | Section 4.1 Database Design | ERD hoặc database design diagram: PostgreSQL, MongoDB, Redis, Elasticsearch |
| Hình 5 | Section 4.2 Caching Strategy | Cache Aside flow: read cache → cache miss → database → write cache |
| Hình 6 | Section 6.2 Reliability Strategy | Reliability/failover diagram: PostgreSQL replication, Redis cluster, circuit breaker |

---

## 1. Project Objective

This project requires students to design a real-world large-scale system using modern system engineering principles.

The selected system is an **E-Commerce Marketplace Platform**, similar to Shopee, designed for the Vietnamese market. The platform supports multiple sellers, many buyers, large product catalogs, high-traffic events such as flash sales, online payment, shipping integration, and analytics dashboards.

The system must demonstrate:

- Reliability
- Scalability
- Maintainability
- Clear system design thinking
- End-to-end data flow
- Trade-off analysis
- Capacity estimation
- Technology justification

This is not only a simple CRUD application or Big Data dashboard. It is a complete data-intensive system that combines:

```text
Web / Mobile Client
+ API Gateway
+ Microservices
+ PostgreSQL
+ MongoDB
+ Redis
+ Elasticsearch
+ Kafka
+ Flink / Spark
+ ClickHouse
+ External Payment / Shipping Services
```

The goal is to design a system that can handle real-world marketplace traffic, especially during peak events such as flash sales, while keeping critical data such as orders and payments consistent.

---

## 2. Scope of the System

Students must design a system that includes all required layers.

### 2.1 Core Components — Mandatory

| Layer | Components |
|---|---|
| Client Layer | Buyer Web, Buyer Mobile, Seller Portal, Admin Panel |
| Application Layer | User Service, Product Catalog Service, Cart Service, Order Service, Payment Service, Inventory Service, Shipping Service, Notification Service |
| Data Layer | PostgreSQL, MongoDB, Redis, Elasticsearch, S3 |
| Data Processing Layer | Kafka, Flink, Spark, ClickHouse |
| Integration Layer | API Gateway, Kafka event bus, external payment/shipping APIs |
| Analytics / Output Layer | Real-time dashboard, GMV report, seller report, system metrics |

This system follows the idea of a **composite data system**:

```text
Database + Cache + Search + Message Queue + Stream Processing + Analytics
```

Instead of using only one database, the system uses different technologies for different requirements:

- PostgreSQL for transactional data.
- MongoDB for flexible product catalog.
- Redis for cache, cart, session, and hot inventory counters.
- Elasticsearch for fast search.
- Kafka for event-driven communication.
- ClickHouse for analytics.

---

## 3. Project Tasks

# Task 1 — Business Problem Definition

## 3.1 Selected Real-World System

The selected system is an **E-Commerce Marketplace Platform** for Vietnam.

The platform allows:

- Buyers to search products, view product details, add items to cart, place orders, pay online, and track delivery.
- Sellers to create products, manage inventory, process orders, and view sales reports.
- Admins to approve products, verify sellers, manage reports, and monitor system health.

The system integrates with external services:

- Payment providers: Momo, VNPay.
- Shipping providers: GHN, GHTK.

---

## 3.2 Users

| User Type | Description | Main Activities |
|---|---|---|
| Buyer | End customer | Search, browse, add cart, checkout, pay, track order |
| Seller | Merchant on marketplace | Create product, update stock, manage orders, view revenue |
| Admin | Platform operator | Approve product, verify seller, handle complaints |
| External Payment Provider | Momo, VNPay | Process payment and send webhook |
| External Shipping Provider | GHN, GHTK | Create shipment and update delivery status |

---

## 3.3 Pain Points

The main pain points in a real marketplace system are:

1. **High traffic during flash sales**
   - Many users access the same products at the same time.
   - Product detail pages and inventory APIs can be overloaded.

2. **Risk of overselling**
   - If inventory is not controlled correctly, sellers may sell more products than available stock.

3. **Payment and order consistency**
   - Payment success, order status, and inventory update must be consistent.
   - Duplicate payment or wrong order status must be avoided.

4. **Slow product search**
   - A large product catalog requires fast full-text search and filtering.

5. **External service failures**
   - Momo, VNPay, GHN, GHTK may be unavailable or slow.
   - The system must not collapse when one external provider fails.

6. **Large data volume**
   - Product images, order history, logs, and analytics events grow quickly.

---

## 3.4 Why Scale Matters

This system needs distributed and large-scale design because:

- There may be millions of buyers and hundreds of thousands of sellers.
- Product catalog can reach tens or hundreds of millions of products.
- Flash sale events can create sudden traffic spikes.
- Search, cart, order, payment, and inventory services have different scalability requirements.
- A single database cannot efficiently handle all workloads.
- External payment and shipping APIs introduce latency and failure risks.

Therefore, the system must be designed with:

- Load balancing.
- Horizontal scaling.
- Caching.
- Database separation.
- Message queues.
- Replication.
- Failover.
- Observability.

---

# Task 2 — Requirements Engineering

## 4.1 Functional Requirements

### 4.1.1 User Service

The system must support:

- User registration.
- User login/logout.
- JWT authentication.
- User profile management.
- Address management.
- Role-based access control for buyer, seller, and admin.

Example APIs:

```http
POST /api/v1/auth/register
POST /api/v1/auth/login
GET  /api/v1/users/me
PUT  /api/v1/users/me
```

---

### 4.1.2 Product Catalog Service

The system must support:

- Seller creates product.
- Seller updates product information.
- Buyer views product detail.
- Product supports images, variants, attributes, tags.
- Product status: draft, active, disabled, moderated.

Example APIs:

```http
POST   /api/v1/seller/products
PUT    /api/v1/seller/products/{product_id}
GET    /api/v1/products/{product_id}
DELETE /api/v1/seller/products/{product_id}
```

---

### 4.1.3 Search Service

The system must support:

- Search by keyword.
- Filter by category, price, rating, seller.
- Sort by relevance, price, newest, sold count.
- Use Elasticsearch for full-text search.

Example API:

```http
GET /api/v1/products/search
```

Input example:

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

---

### 4.1.4 Cart Service

The system must support:

- Add item to cart.
- Remove item from cart.
- Update quantity.
- Calculate subtotal.
- Store cart temporarily in Redis.

Example APIs:

```http
POST /api/v1/cart/items
PUT  /api/v1/cart/items/{product_id}
DELETE /api/v1/cart/items/{product_id}
GET  /api/v1/cart
```

---

### 4.1.5 Order Service

The system must support:

- Create order.
- Track order status.
- View order history.
- Cancel order if allowed.
- Store order and order items in PostgreSQL.

Example APIs:

```http
POST /api/v1/orders
GET  /api/v1/orders/{order_id}
GET  /api/v1/users/me/orders
POST /api/v1/orders/{order_id}/cancel
```

---

### 4.1.6 Payment Service

The system must support:

- Create payment request.
- Redirect or call external payment provider.
- Receive webhook from Momo/VNPay.
- Verify payment signature.
- Update payment status.
- Publish payment events.

Example APIs:

```http
POST /api/v1/payments
POST /api/v1/payments/webhook/momo
POST /api/v1/payments/webhook/vnpay
GET  /api/v1/payments/{payment_id}
```

Important rule:

> Payment amount must be calculated from server-side order data, not from client input.

---

### 4.1.7 Shipping Service

The system must support:

- Create shipping order.
- Get shipping fee.
- Receive shipping status update.
- Update delivery status.
- Integrate with GHN/GHTK.

Example APIs:

```http
POST /api/v1/shipments
GET  /api/v1/shipments/{shipment_id}
POST /api/v1/shipping/webhook/ghn
```

---

### 4.1.8 Notification Service

The system must support:

- Send email, SMS, or push notification.
- Notify order created, payment success, shipment created, delivery updated.
- Consume events from Kafka.

---

### 4.1.9 Admin Service

The system must support:

- Approve/reject products.
- Verify seller.
- Handle reports.
- View system metrics.
- Manage categories and policies.

---

## 4.2 Non-Functional Requirements

Non-functional requirements are critical in this system because the marketplace must handle high traffic, external failures, and data consistency.

| Requirement | Target | Explanation |
|---|---|---|
| Scalability | Normal peak 5K RPS, flash sale up to 50K RPS | System must handle traffic spikes |
| Reliability | RPO < 1 minute, RTO < 5 minutes | Reduce data loss and downtime |
| Availability | 99.9% uptime | Marketplace must be available most of the time |
| Consistency | Strong for payment/order, eventual for catalog/search | Different data has different consistency needs |
| Latency | API P95 < 200ms, Search P95 < 500ms | Users leave if the system is slow |
| Security | JWT, input validation, webhook signature verification | Protect users and transactions |
| Observability | Logs, metrics, tracing, alerting | Detect and fix issues quickly |
| Maintainability | Microservices by domain | Easier to develop and scale |
| Cost efficiency | Cache and CDN reduce database load | Lower infrastructure cost |

---

## 4.3 Out-of-Scope

The following parts are not deeply implemented in this design:

- Warehouse Management System.
- AI recommendation engine.
- Payment processor internals.
- Live streaming shopping.
- Complex ML fraud detection.
- Dynamic pricing engine.
- Full accounting system.
- Deep seller finance settlement system.

---

# Task 3 — High-Level System Design

## 5.1 Architecture Diagram — Mandatory

<!-- IMAGE PLACEHOLDER 1 -->

<!-- NOTE: Chèn Hình 1 tại đây. -->
<!-- Nội dung hình nên có: Client Layer → CDN → Load Balancer/API Gateway → Application Services → Data Layer → Kafka → Analytics/Observability. -->
<!-- Hình cần thể hiện rõ: Load Balancer, Application Servers, Databases, Cache, Message Queue, External Services. -->

Architecture layers:

1. **Client Layer**
   - Buyer Web.
   - Buyer Mobile.
   - Seller Portal.
   - Admin Panel.

2. **Edge Layer**
   - CDN.
   - WAF.
   - Static asset caching.

3. **Gateway Layer**
   - Load Balancer.
   - API Gateway.
   - Authentication.
   - Rate limiting.
   - Routing.

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
   - Admin Service.

5. **Data Layer**
   - PostgreSQL.
   - MongoDB.
   - Redis.
   - Elasticsearch.
   - S3.

6. **Integration Layer**
   - Kafka event bus.
   - External payment APIs.
   - External shipping APIs.

7. **Analytics / Observability Layer**
   - Flink.
   - Spark.
   - ClickHouse.
   - Grafana.
   - Prometheus.
   - ELK/OpenTelemetry.

---

## 5.2 Data Flow

### 5.2.1 Request Flow — User to Response

<!-- IMAGE PLACEHOLDER 2 -->

<!-- NOTE: Chèn Hình 2 tại đây. -->
<!-- Nội dung hình nên mô tả: Buyer → Search → Product Detail → Cart → Checkout → Order → Inventory Reservation → Payment → Notification → Shipping. -->

Normal order flow:

```text
Buyer
  → CDN
  → Load Balancer
  → API Gateway
  → Search Service
  → Product Catalog Service
  → Redis Cache
  → Cart Service
  → Order Service
  → Inventory Service
  → Payment Service
  → External Payment Provider
  → Shipping Service
  → Notification Service
```

Detailed steps:

1. Buyer searches for a product.
2. Search Service returns results from Elasticsearch.
3. Buyer opens product detail.
4. Product Catalog Service checks Redis cache.
   - Cache hit: return data.
   - Cache miss: read MongoDB, then cache the result.
5. Buyer adds item to cart.
6. Cart Service stores cart in Redis.
7. Buyer checks out.
8. Order Service creates order in PostgreSQL.
9. Inventory Service reserves stock in Redis.
10. Payment Service creates payment with Momo/VNPay.
11. Buyer completes payment.
12. Payment provider sends webhook.
13. Payment Service verifies signature and updates payment status.
14. Order status becomes paid.
15. Shipping Service creates shipment.
16. Notification Service sends confirmation to buyer.

---

### 5.2.2 Data Pipeline — Ingestion to Output

<!-- IMAGE PLACEHOLDER 3 -->

<!-- NOTE: Chèn Hình 3 tại đây. -->
<!-- Nội dung hình nên mô tả: Events → Kafka → Flink/Spark → Storage → ClickHouse → Dashboard/Reports. -->

Data pipeline:

```text
User Events / Order Events / Payment Events / Product Events
  → Kafka
  → Flink for real-time processing
  → Spark for batch ETL
  → ClickHouse for analytics
  → Grafana Dashboard / Reports
```

Example data pipeline use cases:

- Real-time GMV dashboard.
- Flash sale monitoring.
- Seller performance report.
- Product popularity ranking.
- Error rate monitoring.
- Payment success rate report.

---

## 5.3 API Design — Basic APIs

The system defines 5 key APIs for the marketplace.

### API 1 — Search Products

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

---

### API 2 — Get Product Detail

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

### API 3 — Add Item to Cart

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

---

### API 4 — Create Order

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

---

### API 5 — Create Payment

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

---

# 4. Deep Dive Design

## 6.1 Database Design

<!-- IMAGE PLACEHOLDER 4 -->

<!-- NOTE: Chèn Hình 4 tại đây. -->
<!-- Nội dung hình nên là ERD hoặc database design diagram. -->
<!-- Cần thể hiện: PostgreSQL cho users/orders/payments, MongoDB cho products, Redis cho cart/cache/session, Elasticsearch cho search. -->

The system uses **polyglot persistence** because different data has different requirements.

### 6.1.1 PostgreSQL

PostgreSQL is used for transactional data:

- Users.
- Orders.
- Payments.
- Sellers.
- Inventory ledger.

Reason:

- ACID transactions.
- Strong consistency.
- Mature indexing.
- Reliable replication.
- Suitable for financial data.

Main tables:

| Table | Purpose |
|---|---|
| users | Store user account information |
| sellers | Store seller profile and verification |
| orders | Store order header and status |
| order_items | Store order item snapshot |
| payments | Store payment status and transaction reference |
| inventory_ledger | Store inventory changes for audit |

---

### 6.1.2 MongoDB

MongoDB is used for flexible product catalog data.

Reason:

- Product schema varies by category.
- Products may have different attributes and variants.
- MongoDB supports horizontal scaling with sharding.
- JSON-like documents are suitable for product metadata.

Example product document:

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
  "status": "active"
}
```

---

### 6.1.3 Redis

Redis is used for:

- Product detail cache.
- Cart.
- Session.
- Rate limiting.
- Hot inventory counters.

Reason:

- Very low latency.
- TTL support.
- Atomic operations.
- Lua scripting.
- Cluster mode.

Important Redis keys:

| Key | Purpose |
|---|---|
| `session:{token}` | Session validation |
| `cart:{user_id}` | Shopping cart |
| `product:{product_id}` | Product detail cache |
| `search:{query_hash}` | Search result cache |
| `inventory:{product_id}` | Hot stock counter |
| `rate_limit:{ip_or_user}` | Rate limiting |
| `idempotency:{request_id}` | Idempotency key |

---

### 6.1.4 Elasticsearch

Elasticsearch is used for product search.

Reason:

- Full-text search.
- Fast filtering and sorting.
- Inverted index.
- Suitable for large product catalog.

---

## 6.2 Caching Strategy

<!-- IMAGE PLACEHOLDER 5 -->

<!-- NOTE: Chèn Hình 5 tại đây. -->
<!-- Nội dung hình nên mô tả Cache Aside pattern: App → Redis → Database. -->
<!-- Cần thể hiện cache hit và cache miss flow. -->

The selected caching strategy is **Cache Aside + Invalidation**.

### Read Path

```text
Application receives request
  → Read Redis
  → Cache hit: return data
  → Cache miss: read database
  → Write result to Redis with TTL
  → Return data
```

### Write Path

```text
Application updates database
  → Delete cache key
  → Next read will miss and reload from database
```

Why delete cache instead of updating cache directly?

- Avoids race condition.
- Avoids stale cache.
- Simpler and more reliable in distributed systems.

### Cache TTL

| Data | TTL |
|---|---:|
| Product detail | 3600s + jitter |
| Cart | 1800s |
| Search result | 300s |
| Session/token | 86400s |
| Inventory count | 30s |
| Rate limit | 60s |
| Flash sale product | 60–300s |

### Thundering Herd Prevention

Thundering herd happens when many cache keys expire at the same time, causing thousands of requests to hit the database together.

Prevention techniques:

- TTL jitter.
- Singleflight/request coalescing.
- Preload hot keys before flash sale.
- Graceful fallback to stale cache if database is overloaded.

---

## 6.3 Message Queue Design — Kafka

Kafka is used as the event bus.

Reason:

- High throughput.
- Durable event log.
- Decouples services.
- Supports retry and replay.
- Suitable for event-driven architecture.

Main Kafka topics:

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

Consumer reliability:

- Consumer groups for parallel processing.
- Offset commit after successful processing.
- Dead Letter Queue for failed events.
- Retry with exponential backoff.
- Idempotent consumers.

---

# 5. Capacity Estimation

## 7.1 Traffic Assumptions

| Metric | Value |
|---|---:|
| MAU | 10,000,000 |
| DAU | 2,000,000 |
| Requests per user per day | 20 |
| Total requests per day | 40,000,000 |
| Normal RPS | ~463 RPS |
| Write ratio | 20% |
| Normal write RPS | ~93 RPS |
| Peak multiplier | 10x |
| Peak RPS | ~4,630 RPS |
| Flash sale target | up to 50,000 RPS |
| Peak concurrent users | ~200,000 |

Calculation:

```text
Total requests/day = DAU × requests/user/day
                   = 2,000,000 × 20
                   = 40,000,000 requests/day

Normal RPS = 40,000,000 / 86,400
           ≈ 463 RPS

Peak RPS = 463 × 10
         ≈ 4,630 RPS
```

---

## 7.2 Storage Estimation

| Data | Calculation | Result |
|---|---|---:|
| Products | 100M × 1KB | ~100GB |
| Orders/day | 500K × 1KB | ~500MB/day |
| Images | 100M × 3 × 200KB | ~60TB |
| Logs/day | 2M × 100 events × 100B | ~20GB/day |
| Total storage growth | Approx. | ~500GB/day, ~180TB/year |

---

## 7.3 Read/Write Ratio

Estimated read/write ratio:

```text
Read : Write = 80 : 20
```

Implication:

- Cache and CDN are critical.
- Read replicas help reduce database load.
- Search must be optimized.
- Writes must be controlled for order, payment, and inventory.

---

# 6. Scalability & Reliability Strategy

## 8.1 Scaling Strategy

### 8.1.1 Vertical Scaling

Vertical scaling means increasing the capacity of a single server.

Examples:

- More CPU.
- More RAM.
- Faster disk.
- Larger database instance.

Advantages:

- Simple.
- No application changes required.

Disadvantages:

- Expensive.
- Limited by hardware.
- Single point of failure risk remains.

Vertical scaling is useful for:

- Initial deployment.
- Small services.
- Database primary during early stage.

---

### 8.1.2 Horizontal Scaling

Horizontal scaling means adding more servers.

Examples:

- More application instances.
- More API Gateway instances.
- More Search Service instances.
- More Kafka brokers.
- More Redis nodes.

Advantages:

- Better scalability.
- Better fault tolerance.
- Can scale specific services independently.

Disadvantages:

- More operational complexity.
- Requires load balancing.
- Requires service discovery and monitoring.

Application services are stateless, so they can be horizontally scaled easily.

```text
More requests
  → Load Balancer distributes traffic
  → Auto Scaling adds more instances
  → Health check removes unhealthy instances
```

---

## 8.2 Load Balancing

Load balancing is used to distribute traffic across multiple service instances.

Used at:

- CDN layer.
- API Gateway layer.
- Application service layer.
- Database read layer.

Benefits:

- Avoids overload on one server.
- Improves availability.
- Supports horizontal scaling.
- Enables rolling deployment.

---

## 8.3 Reliability Strategy

<!-- IMAGE PLACEHOLDER 6 -->

<!-- NOTE: Chèn Hình 6 tại đây. -->
<!-- Nội dung hình nên mô tả reliability/failover: PostgreSQL primary-replica, Redis cluster, Kafka replication, circuit breaker cho payment/shipping provider. -->

### 8.3.1 PostgreSQL Replication and Failover

PostgreSQL uses primary-replica architecture.

```text
Primary Database
  → receives writes
  → replicates to replicas

Replica Databases
  → serve read traffic
  → can be promoted if primary fails
```

Failover flow:

```text
Primary fails
  → HAProxy detects failure
  → Replica promoted to primary
  → Application reconnects to new primary
  → RTO < 5 minutes
```

Backup strategy:

- Daily full backup to S3.
- Hourly incremental backup.
- Point-in-time recovery if possible.

---

### 8.3.2 Redis Cluster

Redis Cluster is used for:

- Cache.
- Cart.
- Session.
- Rate limiting.
- Inventory hot counters.

Reliability:

- Primary-replica per shard.
- Hot keys monitoring.
- Preload flash sale keys.
- Critical counters also mirrored to PostgreSQL inventory ledger.

---

### 8.3.3 MongoDB Sharding

MongoDB is used for product catalog.

Scaling:

- Sharding by `seller_id` or hash-based shard key.
- Multiple shards for horizontal scaling.
- `mongos` routes queries.

Risk:

- Hot shard if one seller has too much traffic.

Mitigation:

- Use hashed shard key.
- Split large sellers into sub-keys.
- Monitor shard size and query distribution.

---

### 8.3.4 Circuit Breaker

Circuit breaker is used for external providers such as Momo, VNPay, GHN, GHTK.

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

Benefits:

- Prevents external failure from crashing the whole system.
- Avoids thread/connection exhaustion.
- Gives fast feedback to users.

---

### 8.3.5 Retry and Idempotency

Retry is used carefully.

Safe to retry:

- Search.
- Product detail.
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
- Use Dead Letter Queue.

---

# 7. Technology Justification

## 9.1 Why Microservices?

Microservices are used because the marketplace has clearly separated domains:

- User.
- Product catalog.
- Cart.
- Order.
- Payment.
- Inventory.
- Shipping.
- Notification.
- Analytics.

Advantages:

- Each domain can scale independently.
- Easier deployment.
- Better fault isolation.
- Teams can work independently.
- Different databases can be used per service.

Trade-off:

- More operational complexity.
- Need service discovery.
- Need monitoring and tracing.
- Distributed transactions are harder.

---

## 9.2 Why React and Flutter?

| Need | Technology | Reason |
|---|---|---|
| Web buyer/seller/admin | React | Fast UI development, large ecosystem |
| Mobile buyer app | Flutter | Cross-platform iOS/Android, good performance |

---

## 9.3 Why CDN / CloudFront?

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

---

## 9.4 Why API Gateway / Kong?

Kong is used for:

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

- Gateway can become a bottleneck.
- Must horizontally scale and monitor it.

---

## 9.5 Why Redis?

Redis is used for:

- Cache.
- Cart.
- Session.
- Rate limiting.
- Hot inventory counters.

Benefits:

- Very low latency.
- TTL support.
- Atomic operations.
- Lua scripting.
- Cluster mode.

Trade-off:

- RAM cost.
- Cache invalidation complexity.
- Not suitable as source of truth for money/order data.

---

## 9.6 Why PostgreSQL?

PostgreSQL is used for:

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

## 9.7 Why MongoDB?

MongoDB is used for:

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

## 9.8 Why Elasticsearch?

Elasticsearch is used for:

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

## 9.9 Why Kafka?

Kafka is used for:

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

## 9.10 Why Flink, Spark, ClickHouse?

| Need | Technology | Reason |
|---|---|---|
| Real-time stream processing | Flink | Low-latency event processing |
| Batch ETL | Spark | Mature batch processing |
| Analytics warehouse | ClickHouse | Fast columnar aggregation |

Why not use PostgreSQL for analytics?

- PostgreSQL is optimized for OLTP.
- ClickHouse is optimized for OLAP and large aggregation.

---

# 8. Conclusion

The final design is a scalable and reliable marketplace system for the Vietnam market. It uses microservices to separate business domains, polyglot persistence to match each data requirement, Redis and CDN to reduce latency and database load, Kafka to decouple services, and PostgreSQL to protect order/payment consistency.

The key design principle is:

> Keep the critical path safe and consistent, while moving non-critical work to async event-driven processing.

This allows the system to support normal marketplace traffic and handle flash sale peak traffic without collapsing the database or external services.

Before final submission, remember to replace all image placeholders with actual diagrams:

- Hình 1: Architecture Diagram.
- Hình 2: Request Flow.
- Hình 3: Data Pipeline.
- Hình 4: Database Design / ERD.
- Hình 5: Caching Strategy.
- Hình 6: Scalability & Reliability Diagram.