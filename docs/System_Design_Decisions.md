# JobBoard Pro — System Design Decisions

## 1. Why Microservices?
Chose microservices over a monolith to demonstrate independent scaling, deployment, and fault isolation. Each service (Auth, Candidate, Job, Application, AI Matching, Notification) owns a single business capability, so teams (or in this case, "roles") can work on them independently without stepping on each other.

## 2. DB-per-Service
Each service has its own PostgreSQL database — no shared tables, no cross-service joins. Cross-service data is fetched via Feign clients (synchronous REST calls), not database queries.
**Tradeoff:** Improves loose coupling and independent scaling, but adds network latency and requires handling eventual consistency instead of DB-level transactions.

## 3. Service Discovery & Config
- **Eureka** — services register themselves and discover each other by name, not hardcoded IPs/ports. Makes scaling instances trivial.
- **Config Server** — centralizes configuration (DB URLs, secrets, feature flags) in one Git-backed repo, so config changes don't need code redeploys.

## 4. API Gateway as Single Entry Point
All external traffic (frontend, Postman) goes through Spring Cloud Gateway — never directly to a service. Benefits: centralized JWT validation, routing, and the ability to add rate-limiting/logging in one place instead of duplicating it per service.

## 5. Authentication — JWT
Stateless JWT chosen over session-based auth because it fits microservices well — no shared session store needed. Gateway validates the token once; services trust the token's claims (userId, role) without re-hitting Auth Service every time.

## 6. Idempotency in Application Service
Duplicate job applications are prevented by checking `(candidateId, jobId)` uniqueness before insert, returning `409 Conflict` on repeat. This avoids duplicate records without needing distributed locks — a common interview talking point on idempotent API design.

## 7. Caching Strategy — Redis on Job Search
Job search is the most frequently hit, read-heavy endpoint. Cached in Redis with TTL-based invalidation to reduce DB load on repeated/popular queries (e.g. "Java developer, Bangalore"). Write operations (new job posted) don't invalidate cache immediately — small staleness window is an accepted tradeoff for performance.

## 8. Inter-Service Communication — Feign (Synchronous)
Application Service calls Candidate Service and Job Service via Feign (REST) to enrich application data. Chose synchronous calls for simplicity at this stage; a message queue (Kafka/RabbitMQ) is a noted future upgrade for Notification Service to decouple further.

## 9. Resilience — Circuit Breaker (Resilience4j)
Added basic circuit breaking on Feign calls so a failing Candidate/Job service doesn't cascade failures into Application Service. Falls back gracefully instead of hanging requests.

## 10. Containerization & Deployment Path
Docker Compose first (local, all services + infra in one command) → Jenkins CI/CD → AWS (RDS, S3, EC2, Elastic Beanstalk) → Kubernetes (Minikube). This staged approach mirrors how real teams progress: get it working locally, automate the pipeline, then scale to cloud/orchestration — rather than jumping straight to K8s.

## Key Interview Talking Points
- Why DB-per-service over shared DB (coupling vs. consistency tradeoff)
- How idempotency is enforced without distributed locks
- Caching invalidation strategy and staleness tradeoffs
- EC2 vs Elastic Beanstalk — manual control vs managed scaling
- Synchronous (Feign) vs asynchronous (event-driven) service communication
