
### **Domain 1: Foundations and Core Concepts**
1. **What is System Design?** Goals, constraints, and trade-offs `[Basic]`
2. **Latency vs. Throughput** `[Basic]`
3. **Bandwidth and Capacity Planning** `[Basic]`
4. **SLA, SLO, SLI:** Definitions and implementation `[Basic -> Advanced]`
5. **CAP Theorem** `[Basic -> Expert]`
6. **PACELC Theorem** `[Advanced -> Expert]`
7. **ACID vs. BASE** `[Basic -> Advanced]`
8. **Consistency Models:** Strong, eventual, causal `[Advanced -> Expert]`
9. **Stateful vs. Stateless Architecture** `[Basic -> Advanced]`
10. **Fault Tolerance and Graceful Degradation** `[Advanced]`
11. **Back-of-the-Envelope Estimation** (QPS, storage, bandwidth) `[Basic]`
12. **12-Factor App Principles** `[Basic -> Advanced]`

### **Domain 2: Networking and Communication**
1. [**OSI Model and TCP/IP Fundamentals** ](./OSI-Model-and-TCP-IP-Fundamentals.html)
2. **DNS Resolution:** TTL, Anycast `[Basic -> Advanced]`
3. **HTTP Evolution:** HTTP/1.1 vs. HTTP/2 vs. HTTP/3 (QUIC) `[Basic -> Expert]`
4. **TLS and mTLS Fundamentals** `[Advanced -> Expert]`
5. **WebSockets vs. SSE vs. Long Polling** `[Basic -> Advanced]`
6. **gRPC Design and Streaming Modes** `[Advanced]`
7. **GraphQL Architecture:** Federation, N+1 mitigation `[Advanced]`
8. **REST API Design Patterns:** Versioning, pagination `[Basic -> Advanced]`
9. **API Gateway Patterns:** Routing, auth, throttling `[Advanced]`
10. **Service Mesh Concepts:** Istio, Linkerd `[Expert]`
11. **CDN Architecture and Cache Invalidation** `[Basic -> Advanced]`
12. **Load Balancing:** L4 vs. L7, routing algorithms `[Basic -> Advanced]`
13. **Reverse Proxy vs. Forward Proxy** `[Basic]`
14. **P2P Networking and WebRTC** `[Advanced -> Expert]`
15. **Network Partition and Split-Brain Handling** `[Expert]`

### **Domain 3: Data Storage and Databases**
1. **Relational Database Schema Design and Indexing** `[Basic -> Advanced]`
2. **Query Optimization and Execution Plans** `[Advanced -> Expert]`
3. **Transactions and Isolation Levels** `[Advanced -> Expert]`
4. **Distributed SQL Systems** `[Expert]`
5. **NoSQL Categories:** Document, key-value, wide-column, graph `[Basic -> Advanced]`
6. **Document Stores and Schema Patterns** `[Advanced]`
7. **Key-Value Stores and TTL/Eviction Strategies** `[Basic -> Advanced]`
8. **Wide-Column Stores:** Partitioning and compaction `[Advanced -> Expert]`
9. **Graph Databases and Query Models** `[Advanced]`
10. **Time-Series Database Design** `[Advanced]`
11. **Search Engines and Relevance Architecture** `[Advanced -> Expert]`
12. **Vector Databases and ANN Indexes** `[Expert -> Cutting Edge]`
13. **NewSQL Approaches** `[Expert]`
14. **Replication Models:** Sync/async, leader-follower, multi-master `[Advanced -> Expert]`
15. **Sharding Strategy and Hotspot Mitigation** `[Advanced -> Expert]`
16. **Read Replicas and Query Routing** `[Advanced]`
17. **Partitioning:** Range, hash, list, composite `[Advanced]`
18. **Object Storage Architecture and Access Patterns** `[Basic -> Advanced]`
19. **Data Archival and Storage Tiering** `[Advanced]`
20. **Zero-Downtime Database Migration** `[Advanced -> Expert]`
21. **Polyglot Persistence** `[Advanced]`
22. **Connection Pooling and DB Proxy Patterns** `[Advanced]`

### **Domain 4: Caching**
1. **Cache Fundamentals and Hit Ratio Optimization** `[Basic -> Advanced]`
2. **Cache Placement Layers:** Client/CDN/App/DB `[Basic -> Advanced]`
3. **Caching Strategies:** Cache-aside, read-through, write-through, write-behind `[Advanced]`
4. **Redis Patterns:** Data structures, streams, pub/sub `[Advanced -> Expert]`
5. **Distributed Cache Topology and Hashing** `[Advanced -> Expert]`
6. **Cache Stampede Mitigation** `[Advanced]`
7. **Cache Invalidation Strategies** `[Advanced]`
8. **Probabilistic Data Structures:** Bloom filters, HyperLogLog `[Advanced -> Expert]`
9. **Hot-Key Mitigation** `[Expert]`

### **Domain 5: Scalability**
1. **Vertical vs. Horizontal Scaling** `[Basic]`
2. **Auto-Scaling:** Reactive, predictive, scheduled `[Advanced]`
3. **Stateless Scaling and Session Externalization** `[Basic -> Advanced]`
4. **Read Scaling via Replicas and Caching** `[Advanced]`
5. **Write Scaling via CQRS and Event-Sourcing** `[Advanced -> Expert]`
6. **Consistent Hashing with Virtual Nodes** `[Advanced -> Expert]`
7. **Cell-Based Architecture** `[Expert]`
8. **Bulkhead Isolation and Blast-Radius Control** `[Advanced]`
9. **Backpressure and Flow Control** `[Advanced -> Expert]`
10. **Data Locality and Affinity Routing** `[Expert]`
11. **Multi-Region and Geo-Distributed Architecture** `[Expert]`
12. **Elasticity vs. Scalability** `[Advanced]`

### **Domain 6: Reliability and Availability**
1. **High Availability Patterns:** Active-active, active-passive `[Advanced]`
2. **Failover Mechanisms and Trade-offs** `[Advanced]`
3. **Circuit Breaker Pattern** `[Advanced]`
4. **Retry Strategy with Exponential Backoff and Jitter** `[Basic -> Advanced]`
5. **Timeouts and Deadline Propagation** `[Advanced]`
6. **Liveness/Readiness/Startup Checks** `[Basic -> Advanced]`
7. **Chaos Engineering Principles and Tooling** `[Expert]`
8. **Disaster Recovery:** RTO/RPO planning `[Advanced]`
9. **Idempotency in Distributed Workflows** `[Advanced -> Expert]`
10. **Exactly-Once Semantics Trade-offs** `[Expert]`
11. **Redundancy and Replication for Resilience** `[Advanced]`
12. **Runbooks and Game-Day Operations** `[Advanced]`
13. **Degraded-Mode Operation Design** `[Advanced]`

### **Domain 7: Messaging and Event-Driven Architecture**
1. **Synchronous vs. Asynchronous Communication** `[Basic]`
2. **Queueing Systems (Point-to-Point)** `[Basic -> Advanced]`
3. **Pub/Sub Systems and Fanout** `[Advanced]`
4. **Kafka Architecture:** Partitions, offsets, consumer groups `[Advanced -> Expert]`
5. **Event-Driven Architecture Patterns** `[Advanced]`
6. **Event Sourcing** `[Advanced -> Expert]`
7. **CQRS** `[Advanced -> Expert]`
8. **Event Mesh and Event Streaming Platforms** `[Expert -> Cutting Edge]`
9. **Outbox Pattern** `[Advanced -> Expert]`
10. **Saga Pattern:** Orchestration vs. choreography `[Advanced -> Expert]`
11. **Dead-Letter Queue Strategy** `[Advanced]`
12. **Stream vs. Batch Processing** `[Advanced]`
13. **Schema Evolution and Compatibility** `[Advanced]`
14. **Delivery Guarantees and Exactly-Once Processing** `[Expert]`

### **Domain 8: Microservices and Service Architecture**
1. **Monolith vs. Microservices Decision Framework** `[Basic -> Advanced]`
2. **Service Decomposition Strategies** `[Advanced]`
3. **Inter-Service Communication Patterns** `[Advanced]`
4. **API Gateway and Backend-for-Frontend (BFF)** `[Advanced]`
5. **Service Discovery Patterns** `[Advanced]`
6. **Sidecar and Ambassador Patterns** `[Advanced]`
7. **Distributed Tracing in Service Ecosystems** `[Advanced -> Expert]`
8. **Modular Monolith Architecture** `[Advanced]`
9. **Strangler Fig Migration Pattern** `[Advanced]`
10. **Service Versioning and Compatibility** `[Advanced]`
11. **Contract Testing** `[Advanced]`
12. **Micro-Frontend Architecture** `[Advanced]`
13. **Shared Library Governance in Distributed Teams** `[Advanced]`

### **Domain 9: Security in System Design**
1. **Authentication Patterns and Credential Lifecycle** `[Basic -> Advanced]`
2. **OAuth 2.0 and OpenID Connect** `[Advanced]`
3. **Authorization Models:** RBAC, ABAC, ReBAC `[Advanced -> Expert]`
4. **Enterprise SSO with SAML** `[Advanced]`
5. **Zero Trust Architecture** `[Expert]`
6. **Encryption in Transit and at Rest** `[Basic -> Advanced]`
7. **Secrets Management Patterns** `[Advanced]`
8. **API Security Hardening** `[Advanced]`
9. **DDoS Mitigation Architecture** `[Advanced -> Expert]`
10. **Software Supply-Chain Security (SBOM, Signing)** `[Expert]`
11. **Data Privacy and Compliance Architecture** `[Advanced]`
12. **Audit Logging and Tamper Evidence** `[Advanced]`
13. **Threat Modeling (STRIDE, PASTA)** `[Advanced -> Expert]`
14. **WAF and IDS/IPS Integration** `[Advanced]`

### **Domain 10: Distributed Systems Theory**
1. **Consensus Algorithms:** Paxos and Raft `[Expert]`
2. **Leader Election Strategies** `[Expert]`
3. **Distributed Locks and Coordination Services** `[Expert]`
4. **Logical Clocks and Causality** `[Expert]`
5. **Lamport Timestamps** `[Expert]`
6. **Byzantine Fault Tolerance** `[Expert -> Cutting Edge]`
7. **Two-Phase and Three-Phase Commit** `[Expert]`
8. **Distributed Transaction Models** `[Expert]`
9. **CRDTs for Conflict-Free Replication** `[Expert -> Cutting Edge]`
10. **Gossip Protocols** `[Expert]`
11. **Quorum-Based Read/Write Systems** `[Expert]`
12. **Linearizability vs. Serializability** `[Expert]`
13. **Fallacies of Distributed Computing** `[Basic -> Advanced]`

### **Domain 11: Cloud and Infrastructure**
1. **Cloud Service Models:** IaaS, PaaS, SaaS, FaaS `[Basic]`
2. **Shared Responsibility Model** `[Basic -> Advanced]`
3. **VPC/Subnet/Routing/Security Architecture** `[Advanced]`
4. **Multi-Cloud and Hybrid Patterns** `[Advanced -> Expert]`
5. **Infrastructure as Code (IaC)** `[Advanced]`
6. **Serverless Architecture and Trade-offs** `[Advanced]`
7. **Container Fundamentals** `[Basic -> Advanced]`
8. **Kubernetes Architecture and Workloads** `[Advanced -> Expert]`
9. **Kubernetes Advanced Patterns:** Operators, CRDs `[Expert]`
10. **Service Mesh on Kubernetes** `[Expert]`
11. **Cloud Cost Optimization Design** `[Advanced]`
12. **Landing Zone Architecture** `[Expert]`
13. **FinOps Operating Model** `[Advanced]`
14. **Spot/Preemptible Instance Strategies** `[Advanced]`

### **Domain 12: Data Engineering and Analytics**
1. **Data Warehouse vs. Data Lake vs. Lakehouse** `[Basic -> Advanced]`
2. **ETL vs. ELT** `[Advanced]`
3. **Batch Processing Architecture** `[Advanced]`
4. **Stream Processing Architecture** `[Advanced -> Expert]`
5. **Lambda Architecture** `[Expert]`
6. **Kappa Architecture** `[Expert]`
7. **Data Mesh Principles** `[Expert -> Cutting Edge]`
8. **Metadata, Catalog, and Governance** `[Advanced]`
9. **CDC (Change Data Capture) Patterns** `[Advanced -> Expert]`
10. **Data Quality and Lineage** `[Advanced]`
11. **Schema Registry and Data Contracts** `[Advanced -> Expert]`
12. **Columnar Formats:** Parquet, ORC, Arrow `[Advanced]`
13. **OLAP Engines and Analytical Serving** `[Expert]`
14. **Feature Stores for ML Systems** `[Expert -> Cutting Edge]`

### **Domain 13: Observability and SRE**
1. **Logs, Metrics, and Traces Fundamentals** `[Basic -> Advanced]`
2. **Structured Logging Architecture** `[Advanced]`
3. **Metrics Collection and Dashboard Strategy** `[Advanced]`
4. **Distributed Tracing and Context Propagation** `[Advanced -> Expert]`
5. **Alerting Strategy and On-Call Design** `[Advanced]`
6. **OpenTelemetry Deep Dive** `[Expert]`
7. **SRE:** Error budgets and toil reduction `[Advanced]`
8. **Synthetic Monitoring and Canary Checks** `[Advanced -> Expert]`
9. **Continuous Profiling** `[Expert -> Cutting Edge]`
10. **AIOps for Anomaly Detection and Auto-remediation** `[Cutting Edge]`

### **Domain 14: CI/CD and Delivery**
1. **Deployment Strategies:** Rolling, canary, blue-green, shadow `[Advanced]`
2. **CI/CD Pipeline Architecture** `[Basic -> Advanced]`
3. **Feature Flag Architecture** `[Advanced]`
4. **Branching Strategies:** Trunk-based vs. Gitflow `[Basic -> Advanced]`
5. **GitOps Delivery Model** `[Advanced -> Expert]`
6. **Supply Chain Controls in CI/CD** `[Expert]`
7. **Test Strategy Portfolio** `[Advanced]`
8. **Database Migration in Release Pipelines** `[Advanced]`
9. **Artifact Versioning and Promotion** `[Advanced]`

### **Domain 15: Real-Time and Streaming Systems**
1. **Real-time vs. Near-real-time Trade-offs** `[Basic -> Advanced]`
2. **Chat System Architecture** `[Advanced]`
3. **Notification Architecture:** Multi-channel fanout `[Advanced]`
4. **Presence and Typing Indicators at Scale** `[Advanced -> Expert]`
5. **Live Video Streaming Design** `[Expert]`
6. **Collaborative Editing (OT vs. CRDT)** `[Expert -> Cutting Edge]`
7. **Geospatial / Ride-Hailing Systems** `[Expert]`
8. **Low-Latency Trading System Design** `[Expert -> Cutting Edge]`
9. **IoT Ingestion and Edge Processing** `[Expert]`

### **Domain 16: AI and ML Systems**
1. **ML Lifecycle Architecture:** Train, evaluate, serve `[Advanced]`
2. **Inference Patterns:** Online, batch, stream `[Advanced]`
3. **Feature Stores and Consistency** `[Expert]`
4. **MLOps:** Model registry, lineage, drift monitoring `[Expert]`
5. **Experimentation and A/B Testing Platforms** `[Advanced -> Expert]`
6. **Shadow Deployments and Canaries for ML** `[Expert]`
7. **LLM Inference Infrastructure** `[Expert -> Cutting Edge]`
8. **Retrieval-Augmented Generation (RAG) Architecture** `[Expert -> Cutting Edge]`
9. **Agentic System Orchestration and Tool Use** `[Cutting Edge]`
10. **Vector Search at Scale** `[Expert -> Cutting Edge]`
11. **Responsible AI Controls** `[Advanced -> Expert]`
12. **Multi-modal System Architecture** `[Cutting Edge]`

### **Domain 17: API and Developer Experience**
1. **REST Resource Modeling and HTTP Semantics** `[Basic]`
2. **API Versioning Lifecycle** `[Advanced]`
3. **Pagination Patterns:** Cursor vs. offset `[Advanced]`
4. **GraphQL Federation** `[Expert]`
5. **Webhook Platform Design** `[Advanced]`
6. **SDK Design and Dev Portal Architecture** `[Advanced]`
7. **OpenAPI/AsyncAPI Contract-First Design** `[Advanced]`
8. **Rate Limiting Algorithms** `[Advanced]`
9. **API Monetization and Quota Systems** `[Advanced]`
10. **API Governance:** Internal vs. external `[Advanced]`

### **Domain 18: Frontend and Mobile Architecture**
1. **SPA vs. SSR vs. SSG vs. ISR** `[Basic -> Advanced]`
2. **Micro-Frontend Patterns** `[Advanced -> Expert]`
3. **State Management Strategies** `[Advanced]`
4. **PWA and Offline-First Architecture** `[Advanced]`
5. **Web Performance Architecture (Core Web Vitals)** `[Advanced]`
6. **Mobile Architecture Patterns** `[Advanced]`
7. **Cross-Platform vs. Native Trade-offs** `[Basic -> Advanced]`
8. **Offline Sync and Conflict Resolution** `[Expert]`
9. **Frontend Observability and RUM** `[Advanced]`

### **Domain 19: Enterprise Architecture Patterns**
1. **Enterprise Integration Patterns** `[Advanced -> Expert]`
2. **Domain-Driven Design (DDD)** `[Advanced -> Expert]`
3. **Hexagonal Architecture** `[Advanced]`
4. **Clean Architecture** `[Advanced]`
5. **TOGAF Overview and Adaptation** `[Advanced]`
6. **Architecture Decision Records (ADRs)** `[Basic -> Advanced]`
7. **Architecture Fitness Functions** `[Expert]`
8. **Platform Engineering and Internal Developer Platforms (IDPs)** `[Expert]`
9. **Technical Debt Management Frameworks** `[Advanced]`
10. **API-First Enterprise Strategy** `[Advanced]`
11. **Team Topologies and Conway’s Law** `[Advanced]`
12. **Evolutionary Architecture** `[Expert]`

### **Domain 20: Classic System Design Case Studies**
1. **URL Shortener Platform**
2. **Web Crawler Platform**
3. **Search Engine Architecture**
4. **Social Feed/Timeline System**
5. **Ride-Hailing Platform (Uber/Lyft)**
6. **Video Streaming Platform (Netflix/YouTube)**
7. **Messaging/Chat Platform (WhatsApp/Slack)**
8. **Payments and Ledger System**
9. **File Storage and Sync Platform (Dropbox/Google Drive)**
10. **E-commerce Platform (Amazon)**
11. **Recommendation Engine**
12. **Notification Platform**
13. **Distributed Rate Limiter**
14. **Ad Serving Platform**
15. **Distributed Message Queue**
16. **Distributed Key-Value Store**
17. **Global CDN Architecture**
18. **Code Collaboration Platform (GitHub)**
19. **Booking and Inventory Platform (Airbnb/Expedia)**
20. **Email Platform**

### **Domain 21: Cutting-Edge and Emerging Topics**
1. **eBPF for Networking and Observability**
2. **Server-Side WebAssembly (Wasm) and Edge Compute**
3. **Confidential Computing and Trusted Execution Environments (TEEs)**
4. **Quantum-Resistant Cryptography**
5. **AI-Native Infrastructure Design**
6. **Self-Healing / Autonomic Systems**
7. **Unified Data and AI Platform Patterns**
8. **Multi-Agent AI Orchestration Platforms**
9. **Event-Driven Serverless at Edge Scale**
10. **Programmable Data Planes and SmartNICs**
11. **Sustainable and Carbon-Aware Architecture**
12. **Decentralized Identity and Verifiable Credentials**
