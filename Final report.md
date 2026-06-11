1. Project Objective
This project requires students to design a real-world large-scale system using modern system engineering principles.
The system must demonstrate:
Reliability, scalability, and maintainability
Clear system design thinking (requirements → architecture → trade-offs)
End-to-end data flow (not just analytics)
👉 Không còn chỉ là “Big Data system”, mà là:
“A complete data-intensive system solving a real-world problem at scale.”

2. Scope of the System
Students must design a system that includes ALL layers:
2.1 Core Components (Mandatory)
Client Layer
Web / Mobile / API users
Application Layer
Business logic
APIs
Data Layer
Database(s)
Storage strategy
Data Processing Layer
Batch / Stream / Real-time
Integration Layer
Message queue / event-driven / APIs
Analytics / Output Layer
Dashboard / AI / reports

👉 Đây chính là mô hình “composite data system” trong DDIA:
DB + cache + stream + search → 1 system hoàn chỉnh 

3. Project Tasks
Task 1 – Business Problem Definition (15%)
Select a real-world system
e.g., Ride-hailing, Smart traffic, E-commerce, Education platform
Identify:
Users
Pain points
Why scale matters
👉 Phải trả lời được:
“Why do we need a distributed / large-scale system here?”

Task 2 – Requirements Engineering (20%)
2.1 Functional Requirements
What the system does
Core features (API level)
2.2 Non-functional Requirements (CRITICAL)
(Bắt buộc theo DDIA)
Scalability (traffic growth)
Reliability (fault tolerance)
Availability (uptime)
Consistency (strong vs eventual)
Latency (performance)
👉 Đây là phần sinh viên hay làm yếu → giờ nâng cấp thành trọng tâm.

2.3 Out-of-scope
Những gì KHÔNG làm (rất quan trọng trong system design)

Task 3 – High-Level System Design (25%)
Sinh viên phải vẽ:
3.1 Architecture Diagram (BẮT BUỘC)
Bao gồm:
Load balancer
Application servers
Database(s)
Cache
Message queue
External services
👉 Giống approach từ Alex Xu:
Start simple → scale out

3.2 Data Flow
Request flow (user → system → response)
Data pipeline (ingestion → processing → storage → output)

3.3 API Design (basic)
Define 3–5 key APIs
Input/output format

4. Deep Dive Design (20%)
Chọn 1–2 phần quan trọng nhất để đi sâu:
Ví dụ:
Database design (SQL vs NoSQL)
Caching strategy
Message queue (Kafka vs RabbitMQ)
Stream processing
AI/analytics module
👉 Phải có:
Design choice
Trade-offs

5. Capacity Estimation (10%)
Theo đúng tinh thần Alex Xu:
Number of users
Requests per second
Data size per day
Storage after 1 year
👉 Đây là phần trước chưa có → thêm vào để đúng “System Engineering”.

6. Scalability & Reliability Strategy (10%)
Sinh viên phải trả lời:
6.1 Scaling
Vertical vs Horizontal scaling
Load balancing
6.2 Reliability
Replication
Failover
Backup
👉 Mapping trực tiếp từ DDIA:
Fault tolerance
Distributed system issues

7. Technology Justification (Bonus / 5%)
Why Kafka?
Why MongoDB?
Why Redis?

