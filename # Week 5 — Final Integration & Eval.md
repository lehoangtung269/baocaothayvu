# Week 5 — Final Integration & Evaluation
## E-Commerce Marketplace Platform — Shopee-like, Việt Nam, Multi-seller B2C

---

## 1. Presentation Structure

### What — Deliver gì?
Week 5 tập trung hoàn thiện **Final Integration & Evaluation**, gồm 4 phần chính:

1. **Final Architecture Refinement**
   - Cải thiện kiến trúc dựa trên feedback từ các tuần trước.
   - Làm rõ luồng end-to-end từ Buyer → API Gateway → Services → Data → External APIs.
   - Bổ sung Inventory Reservation, Idempotency, Outbox Pattern, Observability, Circuit Breaker.

2. **End-to-End Scenarios**
   - **Normal flow**: người dùng search → add cart → create order → payment success → notify → ship.
   - **Peak traffic flow**: flash sale 10× traffic, CDN/cache/autoscaling/Kafka bảo vệ backend.
   - **Failure case**: payment provider timeout hoặc PostgreSQL primary failure, hệ thống xử lý bằng circuit breaker, retry, failover.

3. **Technology Justification**
   - Giải thích vì sao chọn từng công nghệ.
   - Trade-off giữa **cost vs performance**.

4. **Reflection**
   - Những điểm còn hạn chế.
   - Những phần sẽ cải thiện nếu có thêm thời gian.

### When — Deliver khi nào?
- Deliver trong **Week 5 Final Presentation**.
- Đây là phần tổng kết toàn bộ hệ thống sau Week 1 → Week 4.

### Why — Vì sao chọn artifacts này?
- Tutor yêu cầu: **“Use models rather than words”**, nên cần có diagram và scenario flow.
- Week 5 cần thể hiện **clear storytelling**: hệ thống hoạt động thế nào trong thực tế, khi đông người dùng thì xử lý ra sao, khi lỗi thì phục hồi thế nào.
- Technology justification giúp chứng minh lựa chọn công nghệ không phải chọn ngẫu nhiên, mà dựa trên trade-off.

### How Much — Planned vs Spent
| Hạng mục | Planned | Spent | Gap | Action |
|---|---:|---:|---:|---|
| Final architecture refinement | 2h | 1.5h | 0.5h | Dùng diagram để tóm tắt nhanh |
| End-to-end scenarios | 2h | 1.5h | 0.5h | Chuẩn bị 3 flow: Normal/Peak/Failure |
| Technology justification | 1.5h | 1h | 0.5h | Dùng bảng cost/performance |
| Reflection | 1h | 0.5h | 0.5h | Tập trung vào limitation và future improvement |

---

# 2. Final Architecture Refinement

## 2.1 Feedback từ các tuần trước

Từ các diagram và phần trình bày Week 1 → Week 4, kiến trúc cuối cùng cần cải thiện các điểm sau:

| Feedback / Vấn đề | Cải thiện trong Week 5 |
|---|---|
| Kiến trúc ban đầu hơi rộng, cần kể rõ luồng chính | Bổ sung **End-to-End Scenario** cho Normal, Peak, Failure |
| Cần làm rõ cách tránh oversell trong flash sale | Bổ sung **Inventory Reservation Service** và Redis atomic operation |
| Distributed transaction giữa Order, Payment, Inventory khó | Dùng **Saga Pattern + Kafka Events + Idempotency Key** |
| Nếu payment provider lỗi thì hệ thống có thể bị kéo theo | Dùng **Circuit Breaker + Retry + Timeout** |
| Cần chứng minh lựa chọn công nghệ hợp lý | Bổ sung bảng **Technology Justification: Why + Cost vs Performance** |
| Cần phản biện giới hạn của hệ thống | Bổ sung **Reflection: Limitations + What to Improve** |

---

## 2.2 Final Architecture — Refined

### Final architecture flow

```text
[Buyer Web/Mobile] [Seller Portal] [Admin Panel]
          |
          v
[CDN / CloudFront] + [WAF]
          |
          v
[Load Balancer: Nginx / AWS ALB]
          |
          v
[API Gateway: Kong]
  - JWT auth
  - Rate limiting
  - Routing
  - Request validation
          |
          +----------------------+----------------------+----------------------+
          |                      |                      |
          v                      v                      v
[User/Auth Service]   [Product/Search Service]   [Cart Service]
          |                      |                      |
          v                      v                      v
PostgreSQL Users        MongoDB Catalog          Redis Cart/Session
                        Elasticsearch Search
                        S3 Product Images
          |
          v
[Order Service]
  - Idempotency key
  - Outbox table
  - Order state machine
          |
          +----------------------+----------------------+
          |                      |
          v                      v
[Inventory Reservation]   [Payment Service]
  - Redis atomic DECR      - Momo/VNPay
  - PostgreSQL reservation - Payment webhook
  - TTL reservation        - Idempotency key
          |
          v
[Kafka Event Bus]
  Topics:
  - order_created
  - inventory_reserved
  - payment_success
  - payment_failed
  - order_cancelled
  - shipping_created
          |
          +----------------------+----------------------+----------------------+
          |                      |                      |
          v                      v                      v
[Notification Service]   [Shipping Service]    [Analytics Service]
Email/SMS/Push           GHN/GHTK API          Flink + Spark
          |                                      |
          v                                      v
     User receives                        ClickHouse + Grafana
     status update                        Dashboard & reports
```

---

## 2.3 Các thành phần chính trong kiến trúc cuối cùng

| Component | Vai trò |
|---|---|
| CDN / CloudFront | Cache static files, product images, frontend assets; giảm latency cho người dùng Việt Nam |
| WAF | Chặn request độc hại, bot, SQL injection cơ bản |
| Load Balancer | Phân phối request vào nhiều app instances, health check |
| API Gateway | JWT auth, rate limit, routing, request validation |
| User/Auth Service | Register, login, JWT, user profile |
| Product/Search Service | Product catalog, search, category filter |
| Cart Service | Lưu cart tạm thời trong Redis |
| Order Service | Tạo order, quản lý trạng thái order, idempotency |
| Inventory Reservation Service | Giữ hàng tạm thời để tránh oversell trong flash sale |
| Payment Service | Gọi Momo/VNPay, xử lý webhook, cập nhật payment status |
| Shipping Service | Tạo shipment với GHN/GHTK |
| Notification Service | Gửi email/SMS/push notification |
| Kafka | Event bus để decouple services |
| Redis | Cache, cart, session, inventory reservation, rate limit |
| PostgreSQL | Users, orders, payments, reservations — dữ liệu cần ACID |
| MongoDB | Product catalog — schema linh hoạt theo ngành hàng |
| Elasticsearch | Search nhanh, full-text search, filter theo giá/category |
| S3 | Lưu product images |
| Flink/Spark/ClickHouse | Real-time và batch analytics |
| Grafana | Monitoring, alerting, dashboard |

---

## 2.4 Điểm cải tiến quan trọng nhất

### 1. Inventory Reservation thay vì chỉ “check stock”

Ở Week 2, luồng đặt hàng chỉ có:

```text
Order Service → Redis check inventory → PostgreSQL INSERT order
```

Vấn đề: trong flash sale, nhiều người có thể cùng đặt một sản phẩm cuối cùng. Nếu chỉ check stock rồi insert order, có thể xảy ra **oversell**.

Cải thiện Week 5:

```text
Order created
  → Inventory Reservation Service giữ hàng tạm thời
  → Payment pending
  → Nếu payment success → commit inventory
  → Nếu payment failed/timeout → release inventory
```

Redis atomic operation:

```text
DECR inventory:{product_id}
```

Nếu kết quả < 0 thì rollback:

```text
INCR inventory:{product_id}
Return “sold out”
```

---

### 2. Idempotency Key

Trong thanh toán, user có thể bấm nút nhiều lần hoặc payment webhook gửi lại nhiều lần. Nếu không có idempotency, hệ thống có thể tạo nhiều payment/order.

Giải pháp:

```text
Client gửi idempotency_key với request tạo order/payment.
Server lưu idempotency_key vào PostgreSQL/Redis.
Nếu request trùng key → trả lại response cũ, không tạo lại order/payment.
```

---

### 3. Outbox Pattern

Order Service sau khi tạo order cần publish event sang Kafka. Nếu tạo order thành công nhưng publish Kafka thất bại, hệ thống sẽ mất sự kiện.

Giải pháp:

```text
Trong cùng transaction với việc insert order:
- INSERT into orders
- INSERT into outbox_events
Sau đó worker đọc outbox_events và publish sang Kafka.
```

---

### 4. Saga Pattern

Vì hệ thống dùng nhiều service và nhiều database, không nên dùng distributed transaction trực tiếp.

Thay vào đó dùng Saga:

```text
order_created
→ reserve_inventory
→ initiate_payment
→ if payment_success → confirm_order + create_shipment
→ if payment_failed → cancel_order + release_inventory
```

---

### 5. Observability

Bổ sung monitoring để biết hệ thống đang khỏe hay không:

| Layer | Metric |
|---|---|
| API Gateway | RPS, latency P95, error rate |
| App Services | CPU, memory, request rate, exception count |
| Redis | Hit ratio, memory usage, connection count |
| PostgreSQL | Query latency, replica lag, connection pool |
| Kafka | Consumer lag, topic throughput |
| External APIs | Timeout rate, circuit breaker state |
| Business | Orders/min, payment success rate, cart abandonment |

---

# 3. End-to-End Scenarios

---

## 3.1 Scenario 1 — Normal Flow

### Mục tiêu
Người dùng mua một sản phẩm thành công từ lúc search đến lúc nhận thông báo đơn hàng.

### Flow chi tiết

```text
1. Buyer mở app/web
   → CDN trả frontend assets và images.

2. Buyer search “giày nam”
   → API Gateway → Product/Search Service
   → Elasticsearch trả danh sách sản phẩm.

3. Buyer xem product detail
   → Product Service kiểm tra Redis cache.
   → Nếu cache miss → MongoDB lấy catalog, S3 lấy image.
   → Lưu lại Redis TTL 3600s.

4. Buyer thêm vào cart
   → Cart Service lưu vào Redis key cart:{user_id}.
   → Cart chưa tạo order, chưa trừ tồn kho.

5. Buyer checkout
   → Order Service nhận request với idempotency_key.
   → Kiểm tra JWT, cart, shipping address, payment method.

6. Inventory Reservation
   → Inventory Reservation Service dùng Redis atomic DECR.
   → Nếu còn hàng → giữ hàng tạm thời TTL 10 phút.
   → Nếu hết hàng → trả 400 Sold Out.

7. Tạo order
   → Order Service INSERT order vào PostgreSQL với status PAYMENT_PENDING.
   → INSERT outbox event order_created.

8. Payment initiation
   → Payment Service gọi Momo/VNPay.
   → User chuyển sang payment page/app.

9. Payment webhook
   → Momo/VNPay gọi webhook payment_success.
   → Payment Service kiểm tra chữ ký, amount, order_id.
   → Cập nhật payment status = PAID.

10. Kafka event
   → Payment Service publish payment_success.
   → Order Service nhận event → update order status = PAID.
   → Inventory Service commit reserved stock.
   → Shipping Service tạo shipment với GHN/GHTK.
   → Notification Service gửi email/SMS/push.

11. Buyer xem order status
   → Order Service trả status PAID/SHIPPING.
```

### Kết quả
- Order được tạo đúng một lần.
- Inventory được giữ và commit đúng.
- Payment không bị tính trùng nhờ idempotency.
- User nhận được notification.
- Các service không gọi trực tiếp quá nhiều vào nhau; phần lớn giao tiếp qua Kafka event.

---

## 3.2 Scenario 2 — Peak Traffic / Flash Sale

### Giả định traffic
Từ Week 1:

```text
Normal RPS:       ~463 RPS
Peak RPS:         ~4,630 RPS
Peak concurrent:  ~200,000 users
Read:Write ratio: 80:20
```

Trong flash sale, vấn đề chính là:
- Hàng nghìn người cùng xem một sản phẩm.
- Hàng nghìn người cùng checkout một sản phẩm hot.
- Database có thể bị overload.
- Payment provider có thể timeout.
- Inventory có thể bị oversell.

---

### Peak traffic flow

```text
1. Buyer truy cập flash sale page
   → CDN cache frontend assets và product images.
   → Giảm request về backend.

2. Buyer search/filter sản phẩm flash sale
   → Elasticsearch xử lý search thay vì query MongoDB/PostgreSQL trực tiếp.
   → Search result có thể cache ngắn hạn TTL 30-60s.

3. Buyer xem product detail
   → Redis cache product detail.
   → Nếu hit → trả về ~0.1ms.
   → Nếu miss → MongoDB/S3 load và cache lại.

4. Buyer add to cart
   → Redis lưu cart.
   → Không ghi PostgreSQL ngay.

5. Buyer checkout cùng lúc
   → API Gateway rate limit theo user/IP.
   → Order Service nhận request.
   → Inventory Reservation Service dùng Redis atomic operation.

6. Inventory protection
   → Chỉ người giữ được inventory mới được tiếp tục payment.
   → Nếu hết inventory → trả Sold Out nhanh.
   → PostgreSQL không bị flood bởi quá nhiều request checkout.

7. Kafka buffering
   → Các event order_created, inventory_reserved, payment_success được đẩy vào Kafka.
   → Services xử lý theo tốc độ của chúng.
   → Tránh gọi đồng bộ dây chuyền quá dài.

8. Autoscaling
   → Load Balancer phát hiện CPU >70% trong 5 phút.
   → Auto Scaling thêm instances.
   → Khi traffic giảm, scale in để tiết kiệm chi phí.

9. Database protection
   → PostgreSQL primary nhận writes.
   → Read replicas nhận reads.
   → PgBouncer/HAProxy giới hạn connection pool.
   → Query quan trọng có index.

10. External API protection
   → Payment Service dùng timeout + retry + circuit breaker.
   → Nếu Momo/VNPay chậm, không giữ thread quá lâu.
   → Order vẫn ở trạng thái PAYMENT_PENDING thay vì crash.
```

### Vì sao hệ thống chịu được peak traffic?

| Vấn đề | Giải pháp |
|---|---|
| CDN/origin bị overload | CloudFront cache images/static files |
| App server quá tải | Load Balancer + Auto Scaling |
| Database bị query flood | Redis cache + Elasticsearch + read replicas |
| Oversell | Redis atomic inventory reservation |
| Payment provider chậm | Timeout + retry + circuit breaker |
| Event bị mất | Kafka + Outbox Pattern |
| Request trùng | Idempotency key |

### Target metrics trong peak traffic

| Metric | Target |
|---|---:|
| API P95 latency | <200ms với catalog/cart |
| Search P95 latency | <500ms |
| Availability | 99.9% |
| Cache hit ratio | ~90% |
| RTO | <5 phút |
| RPO | <1 phút với payment/order |
| Peak traffic | 10× normal traffic |

---

## 3.3 Scenario 3 — Failure Case

### Failure case chọn trình bày
Có thể chọn một trong hai failure case sau:

1. **Payment provider timeout** — thực tế dễ xảy ra với Momo/VNPay.
2. **PostgreSQL primary failure** — thể hiện reliability và failover.

Nên trình bày cả hai ngắn gọn, nhưng đi sâu vào **Payment provider timeout** vì liên quan trực tiếp đến order flow.

---

## 3.3.1 Failure Case A — Payment Provider Timeout

### Tình huống
Người dùng đã tạo order, hệ thống gọi Momo/VNPay nhưng payment provider không phản hồi trong 3 giây.

### Flow xử lý lỗi

```text
1. Order Service tạo order status = PAYMENT_PENDING.

2. Payment Service gọi Momo/VNPay.

3. Momo/VNPay timeout sau 3s.

4. Payment Service không cancel order ngay.
   → Đánh dấu payment status = PENDING_RETRY.
   → Ghi log và metric.

5. Circuit Breaker đếm lỗi.
   → Nếu 5 lần lỗi liên tiếp → OPEN.

6. Khi Circuit Breaker OPEN:
   → Không gọi payment provider ngay.
   → Trả user message:
      “Payment gateway is busy. Please retry later.”
   → Order vẫn tồn tại, không mất dữ liệu.

7. Sau 30s:
   → Circuit Breaker chuyển HALF-OPEN.
   → Thử gọi 1 request.
   → Nếu thành công → CLOSED.
   → Nếu thất bại → OPEN lại.

8. Nếu payment webhook đến sau:
   → Payment Service kiểm tra chữ ký và idempotency.
   → Nếu hợp lệ → publish payment_success.
   → Order Service cập nhật order status.
```

### Vì sao không cancel order ngay?
Vì payment provider có thể xử lý thành công ở phía ngân hàng nhưng webhook bị delay. Nếu cancel quá sớm, user có thể bị trừ tiền nhưng order bị hủy.

### Kết quả
- User không mất order.
- Payment không bị tính trùng.
- Hệ thống không bị kéo sập bởi external API.
- Circuit breaker bảo vệ app server khỏi waiting quá lâu.

---

## 3.3.2 Failure Case B — PostgreSQL Primary Failure

### Tình huống
PostgreSQL primary chứa orders/payments bị lỗi.

### Flow xử lý lỗi

```text
1. HAProxy/PgBouncer phát hiện primary không healthy.

2. Writes tạm thời dừng hoặc trả retryable error.

3. Replica khỏe nhất được promote thành primary mới.

4. Application reconnect qua connection pool.

5. Order/Payment Service retry transaction với idempotency.

6. Replica còn lại tiếp tục replication từ primary mới.

7. RTO target <5 phút.
```

### Rủi ro
- Nếu replication async, có thể mất dữ liệu gần nhất.
- Payment cần consistency cao hơn catalog.
- Vì vậy payment replication nên dùng sync hoặc semi-sync.

### Kết quả
- System vẫn phục hồi được.
- Dữ liệu quan trọng được ưu tiên consistency.
- RTO/RPO vẫn nằm trong target.

---

## 3.3.3 Failure Handling Summary

| Failure | Detection | Response | Recovery |
|---|---|---|---|
| Payment timeout | Timeout >3s | Retry, keep order PENDING | Circuit breaker, webhook reconciliation |
| PostgreSQL primary down | Health check fails | Promote replica | RTO <5min |
| Redis down | Connection error | Fallback to PostgreSQL for critical reads | Restore Redis, warm cache |
| Kafka node down | Broker health check | Kafka cluster elects leader | <1min |
| App server down | LB health check fails | Remove instance, ASG replaces | ~2min |
| External shipping API down | Timeout/error rate | Circuit breaker, retry later | Reconcile shipment queue |

---

# 4. Technology Justification

---

## 4.1 Technology Selection Table

| Layer | Technology | Why choose it? | Performance benefit | Cost/trade-off |
|---|---|---|---|---|
| Frontend Web | React | Phổ biến, component-based, dễ tuyển dev | Fast UI, reusable components | Cần quản lý state và bundle size |
| Mobile App | Flutter | Cross-platform iOS/Android | Một codebase cho 2 nền tảng | Performance có thể kém native trong một số case |
| CDN | CloudFront | Edge caching, global network | Giảm latency từ ~500ms xuống ~20ms | Tốn chi phí cache/data transfer |
| Load Balancer | Nginx / AWS ALB | Health check, distribute traffic | Tăng availability | Managed ALB tốn hơn self-host Nginx |
| API Gateway | Kong | Auth, rate limit, routing, plugins | Giảm logic lặp trong services | Cần vận hành/config đúng |
| Cache/Session/Cart | Redis Cluster | Latency ~0.1ms, TTL, atomic ops | Giảm load PostgreSQL | Tốn RAM, cần invalidation strategy |
| Orders/Payments/Users | PostgreSQL | ACID, strong consistency | Phù hợp transaction | Khó scale ngang hơn NoSQL |
| Product Catalog | MongoDB | Flexible schema, horizontal scaling | Phù hợp catalog đa dạng | Query phức tạp cần index/shard design |
| Search | Elasticsearch | Inverted index, full-text search | Search P95 <500ms | Tốn RAM/CPU, vận hành cluster |
| Images | S3 | Cheap object storage, durable | CDN có thể cache S3 objects | Cost theo storage/request |
| Event Bus | Kafka | High throughput, replay, decoupling | Xử lý peak traffic tốt | Vận hành phức tạp hơn RabbitMQ |
| Stream Processing | Apache Flink | Real-time analytics, fraud detection | Low-latency event processing | Learning curve, infra cost |
| Batch Processing | Apache Spark | ETL, daily reports | Xử lý dữ liệu lớn | Batch latency cao hơn streaming |
| Data Warehouse | ClickHouse | Columnar, aggregation nhanh | Dashboard/report nhanh | Không phù hợp transaction |
| Monitoring | Grafana + metrics | Visualization, alerting | Phát hiện lỗi sớm | Cần setup dashboard/metric đúng |
| DB Pooling | PgBouncer/HAProxy | Giới hạn connection, failover | Bảo vệ PostgreSQL | Thêm một layer vận hành |
| Payment | Momo/VNPay | Phù hợp thị trường Việt Nam | Thanh toán local | Phụ thuộc external API |
| Shipping | GHN/GHTK | Phù hợp logistics Việt Nam | Tạo shipment tự động | API có thể timeout/rate limit |

---

## 4.2 Cost vs Performance

### High performance, higher cost

| Component | Cost | Performance |
|---|---|---|
| Managed Kafka | Cao | Throughput cao, replay, reliable event bus |
| Elasticsearch Cluster | Trung bình-cao | Search rất nhanh |
| Managed PostgreSQL | Trung bình-cao | ACID, backup, replica, failover |
| Redis Cluster | Trung bình | Giảm mạnh load DB |
| CloudFront CDN | Theo usage | Giảm latency và origin load |

### Cost optimization strategy

| Strategy | Effect |
|---|---|
| Cache 90% read requests | Giảm request về DB |
| CDN cache images/static files | Giảm bandwidth origin |
| Auto-scaling stateless services | Chỉ trả tiền khi cần |
| Use read replicas only khi cần | Tránh over-provision |
| S3 lifecycle policy | Chuyển ảnh cũ sang cheaper storage class |
| Batch analytics bằng Spark | Giảm chi phí so với xử lý real-time toàn bộ |
| Kafka topic retention hợp lý | Tránh lưu log/event quá lâu |
| Rate limiting | Giảm abuse và chi phí request không cần thiết |

---

## 4.3 Why not use one database for everything?

Không dùng một database cho tất cả vì mỗi loại dữ liệu có yêu cầu khác nhau:

| Data type | Requirement | Best fit |
|---|---|---|
| Users | Secure, relational, consistent | PostgreSQL |
| Orders | ACID, transaction, audit | PostgreSQL |
| Payments | Strong consistency, no data loss | PostgreSQL |
| Product catalog | Flexible schema, variants, attributes | MongoDB |
| Search | Full-text, ranking, filters | Elasticsearch |
| Cart/session | Fast read/write, TTL | Redis |
| Analytics | Aggregation, time-series | ClickHouse |

Nếu dùng một database duy nhất:
- PostgreSQL sẽ phải chứa catalog rất lớn và search kém hiệu quả.
- MongoDB không phù hợp cho payment transaction.
- Redis không phù hợp lưu dữ liệu持久.
- Elasticsearch không phải database chính cho transaction.

Vì vậy, hệ thống dùng **polyglot persistence**.

---

## 4.4 Why Kafka instead of direct API calls?

Direct calls:

```text
Order Service → Payment Service → Shipping Service → Notification Service
```

Vấn đề:
- Nếu một service chậm, toàn bộ chain bị chậm.
- Khó retry.
- Khó scale độc lập.
- Dễ gây cascade failure.

Kafka:

```text
Order Service → Kafka → Payment/Shipping/Notification Services
```

Ưu điểm:
- Decoupling services.
- Event replay khi có bug.
- Consumer xử lý theo tốc độ riêng.
- Giảm coupling và tăng reliability.

Trade-off:
- Kafka phức tạp hơn direct API.
- Cần monitor consumer lag.
- Eventual consistency thay vì synchronous response.

---

## 4.5 Why Redis for inventory reservation?

Trong flash sale, PostgreSQL có thể không đủ nhanh nếu hàng nghìn request cùng checkout.

Redis phù hợp vì:
- Latency ~0.1ms.
- Atomic operations: `DECR`, `INCR`, `SETNX`.
- TTL tự động release reservation.
- Throughput cao hơn PostgreSQL nhiều lần.

Trade-off:
- Redis là in-memory, cần persistence/replication.
- Nếu Redis mất dữ liệu, cần fallback và reconciliation.
- Không dùng Redis làm source of truth cuối cùng cho payment/order.

---

# 5. Reflection

---

## 5.1 What would I improve?

Nếu có thêm thời gian, tôi sẽ cải thiện các phần sau:

### 1. Implement Saga orchestration rõ hơn

Hiện tại hệ thống dùng event-based Saga. Tuy nhiên, trong thực tế cần có orchestration hoặc choreography rõ ràng.

Có thể cải thiện bằng:
- Workflow engine đơn giản.
- Saga state table.
- Compensation action cho từng bước:
  - cancel order
  - release inventory
  - refund payment
  - cancel shipment

---

### 2. Improve inventory accuracy

Hiện tại dùng Redis atomic operation để giữ hàng. Nhưng Redis vẫn có thể mất dữ liệu nếu cluster lỗi.

Cải thiện:
- Redis persistence AOF.
- Inventory reservation cũng lưu vào PostgreSQL.
- Background reconciliation giữa Redis và PostgreSQL.
- Lock product SKU trong thời gian ngắn khi flash sale.

---

### 3. Better shard key design

Hiện tại MongoDB catalog sharding theo `seller_id`.

Vấn đề:
- Nếu một seller rất lớn, có thể tạo hot shard.
- Query theo category có thể phải scatter-gather nhiều shard.

Cải thiện:
- Composite shard key: `category_hash + product_id`.
- Hoặc hash-based sharding.
- Phân tích query pattern trước khi chọn shard key.

---

### 4. Stronger observability

Cần thêm:
- Distributed tracing: OpenTelemetry.
- Correlation ID cho mỗi request.
- Alert theo business metric:
  - payment success rate giảm
  - order_created tăng bất thường
  - consumer lag tăng
  - inventory reservation failure tăng

---

### 5. Load testing trước khi production

Cần chạy:
- k6 hoặc JMeter.
- Test 4,630 RPS.
- Test flash sale scenario.
- Test payment timeout.
- Test PostgreSQL failover.
- Test Redis failover.

---

### 6. Security improvement

Bổ sung:
- WAF rules.
- Rate limit theo user/device/IP.
- Payment webhook signature verification.
- Secret rotation.
- Audit log cho admin/seller actions.
- PII encryption cho user data.

---

## 5.2 Limitations

### Limitation 1 — Distributed system complexity

Hệ thống dùng nhiều service, nhiều database, Kafka, Redis, Elasticsearch. Điều này làm hệ thống mạnh hơn nhưng cũng khó vận hành hơn.

Vấn đề:
- Debug khó hơn monolith.
- Cần observability tốt.
- Cần DevOps maturity.

---

### Limitation 2 — Eventual consistency

Vì dùng Kafka và Saga, một số trạng thái không đồng bộ ngay lập tức.

Ví dụ:
- User đã thanh toán nhưng order status chưa cập nhật ngay.
- Inventory đã reserved nhưng shipping chưa tạo ngay.
- Analytics dashboard có độ trễ.

Trade-off:
- Tăng scalability.
- Giảm synchronous coupling.
- Nhưng cần reconciliation và idempotency.

---

### Limitation 3 — External API dependency

Momo/VNPay/GHN/GHTK là external services. Nếu họ lỗi, hệ thống không thể kiểm soát hoàn toàn.

Giải pháp:
- Circuit breaker.
- Retry with exponential backoff.
- Webhook reconciliation.
- Manual support flow.

---

### Limitation 4 — Cache invalidation risk

Redis giúp tăng performance nhưng có thể trả dữ liệu cũ nếu invalidation không đúng.

Ví dụ:
- Seller đổi giá.
- Product detail vẫn cache trong 1 giờ.
- Buyer thấy giá cũ.

Giải pháp:
- TTL ngắn cho giá/inventory.
- DEL cache khi seller update product.
- Versioned cache key.
- Event-driven invalidation.

---

### Limitation 5 — Cost of managed services

Managed Kafka, Elasticsearch, PostgreSQL, Redis, ClickHouse có thể tốn chi phí cao.

Giải pháp:
- Dùng managed service cho phần quan trọng.
- Tự host hoặc dùng cheaper alternative cho phần ít critical.
- Auto-scaling và lifecycle policy.

---

## 5.3 Lessons learned

- Không nên chọn công nghệ chỉ vì “popular”.
- Mỗi công nghệ nên được chọn dựa trên requirement:
  - ACID → PostgreSQL.
  - Flexible schema → MongoDB.
  - Search → Elasticsearch.
  - Cache/session/cart → Redis.
  - Event streaming → Kafka.
  - Analytics → ClickHouse/Flink/Spark.
- Scalability không chỉ là thêm server.
- Reliability cần có retry, timeout, circuit breaker, idempotency, monitoring.
- Diagram giúp trình bày hệ thống rõ hơn rất nhiều so với chỉ nói bằng text.

---

# 6. Final Presentation Storyline

## Slide 1 — Title
**Week 5: Final Integration & Evaluation**
E-Commerce Marketplace Platform — Shopee-like, Việt Nam.

---

## Slide 2 — What we improved
Nêu 4 cải tiến chính:
- Final architecture refinement.
- Inventory reservation.
- Saga + idempotency + outbox.
- Observability + failure handling.

---

## Slide 3 — Final Architecture
Trình bày diagram final architecture:
- Client → CDN/WAF → Load Balancer → API Gateway.
- Services: User, Product/Search, Cart, Order, Inventory, Payment, Shipping, Notification.
- Data: Redis, PostgreSQL, MongoDB, Elasticsearch, S3, Kafka, ClickHouse.

---

## Slide 4 — Normal Flow
Kể câu chuyện:
“Một buyer search sản phẩm, thêm vào cart, checkout, payment success, order được xác nhận và notification được gửi.”

---

## Slide 5 — Peak Traffic Flow
Nhấn mạnh flash sale:
- 10× traffic.
- CDN/cache giảm load.
- Redis inventory reservation tránh oversell.
- Kafka buffering events.
- Autoscaling thêm instances.

---

## Slide 6 — Failure Case
Chọn payment timeout:
- Payment provider không phản hồi.
- Order vẫn PENDING.
- Circuit breaker mở sau 5 lỗi.
- Retry sau 30s.
- Webhook reconciliation.

---

## Slide 7 — Technology Justification
Trình bày bảng:
- PostgreSQL cho transaction.
- MongoDB cho catalog.
- Redis cho cache/cart/session/inventory.
- Elasticsearch cho search.
- Kafka cho event bus.
- ClickHouse/Flink/Spark cho analytics.

---

## Slide 8 — Cost vs Performance
Nêu trade-off:
- Managed services tốn tiền nhưng giảm vận hành.
- CDN/Redis tiết kiệm DB load.
- Kafka/Elasticsearch tăng performance nhưng tăng complexity.
- Auto-scaling giúp tối ưu chi phí.

---

## Slide 9 — Reflection
Nêu limitation:
- Distributed complexity.
- Eventual consistency.
- External API dependency.
- Cache invalidation risk.
- Cost of managed services.

---

## Slide 10 — What to improve next
- Saga orchestration.
- Better shard key.
- OpenTelemetry tracing.
- Load testing.
- Chaos testing.
- Stronger security.

---

## Slide 11 — Conclusion
Kết luận:

> Hệ thống được thiết kế để xử lý marketplace quy mô lớn tại Việt Nam, với yêu cầu cao về availability, scalability và reliability. Kiến trúc sử dụng polyglot persistence, event-driven communication, caching, autoscaling và circuit breaker để đáp ứng flash sale traffic và giảm rủi ro failure. Tuy vẫn còn limitations về distributed complexity và external dependency, hệ thống có hướng cải thiện rõ ràng bằng Saga orchestration, observability, load testing và chaos testing.

---

# 7. Short Speaking Script

## 7.1 Normal flow script

“Trong normal flow, buyer bắt đầu bằng việc search sản phẩm. Request đi qua CDN, Load Balancer và API Gateway. Product/Search Service dùng Elasticsearch để trả kết quả nhanh. Khi buyer thêm vào cart, dữ liệu được lưu tạm trong Redis. Khi checkout, Order Service tạo order với idempotency key. Inventory Reservation Service giữ hàng bằng Redis atomic operation để tránh oversell. Sau đó Payment Service gọi Momo hoặc VNPay. Khi payment webhook trả về success, hệ thống publish event sang Kafka để Notification, Shipping và Analytics xử lý.”

---

## 7.2 Peak traffic script

“Trong flash sale, traffic có thể tăng 10 lần, khoảng 4,630 RPS. Để bảo vệ backend, CDN cache static files và images, Redis cache product detail và cart, Elasticsearch xử lý search, và Kafka buffering events. Quan trọng nhất là Inventory Reservation dùng Redis atomic operation để chỉ người giữ được hàng mới tiếp tục payment. Nếu hết hàng, hệ thống trả Sold Out nhanh mà không flood PostgreSQL.”

---

## 7.3 Failure case script

“Trong failure case, ví dụ payment provider timeout. Hệ thống không cancel order ngay vì webhook có thể đến muộn. Order vẫn ở trạng thái PAYMENT_PENDING. Payment Service dùng retry, timeout và circuit breaker. Nếu provider lỗi nhiều lần, circuit breaker mở để tránh kéo cả hệ thống xuống. Sau 30 giây, hệ thống thử lại một request. Nếu thành công, circuit breaker đóng lại. Cách này giúp hệ thống vẫn reliable dù external API không ổn định.”

---

## 7.4 Technology justification script

“Chúng em không dùng một database cho tất cả vì mỗi loại dữ liệu có requirement khác nhau. Orders và payments cần ACID nên dùng PostgreSQL. Product catalog cần schema linh hoạt nên dùng MongoDB. Search cần inverted index nên dùng Elasticsearch. Cart, session và inventory reservation cần latency rất thấp nên dùng Redis. Kafka dùng để decouple services và xử lý event trong peak traffic. Đây là polyglot persistence và event-driven architecture.”

---

## 7.5 Reflection script

“Limitation lớn nhất là distributed system complexity. Khi dùng nhiều services và databases, debugging và monitoring khó hơn monolith. Thứ hai là eventual consistency vì nhiều action được xử lý qua Kafka. Thứ ba là external API dependency với payment và shipping providers. Nếu có thêm thời gian, chúng em sẽ cải thiện Saga orchestration, distributed tracing, load testing, chaos testing và shard key design.”