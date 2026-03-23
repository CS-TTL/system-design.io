---
layout: default
title: Topics for System Design
---


[**Companion Youtube Channel**](https://www.youtube.com/@CSAI-TTL)

https://www.youtube.com/@CSAI-TTL

### **Domain 1: Foundations and Core Concepts**
1. **What is System Design?** Goals, constraints, and trade-offs 
2. **Latency vs. Throughput** 
3. **Bandwidth and Capacity Planning** 
4. **SLA, SLO, SLI:** Definitions and implementation 
5. **CAP Theorem** 
6. **PACELC Theorem** 
7. **ACID vs. BASE** 
8. **Consistency Models:** Strong, eventual, causal 
9. **Stateful vs. Stateless Architecture** 
10. **Fault Tolerance and Graceful Degradation** 
11. **Back-of-the-Envelope Estimation** (QPS, storage, bandwidth) 
12. **12-Factor App Principles** 

### **Domain 2: Networking and Communication**

1. [**OSI Model and TCP/IP Fundamentals**](./Domain_2/D2_01_OSI-Model-and-TCP-IP-Fundamentals.md)
2. [**DNS Resolution: TTL, Anycast**](./Domain_2/D2_02_DNS-Resolution-TTL-Anycast.md)
3. [**HTTP Evolution: HTTP/1.1 vs. HTTP/2 vs. HTTP/3 (QUIC)**](./Domain_2/D2_03_HTTP-Evolution-HTTP-1-1-vs-HTTP-2-vs-HTTP-3-QUIC.md)
4. [**TLS and mTLS Fundamentals**](./Domain_2/D2_04_TLS-and-mTLS-Fundamentals.md)
5. [**WebSockets vs. SSE vs. Long Polling**](./Domain_2/D2_05_WebSockets-vs-SSE-vs-Long-Polling.md)
6. [**gRPC Design and Streaming Modes**](./Domain_2/D2_06_gRPC-Design-and-Streaming-Modes.md)
7. [**GraphQL Architecture: Federation, N+1 mitigation**](./Domain_2/D2_07_GraphQL-Architecture-Federation-N-1-mitigation.md)
8. [**REST API Design Patterns: Versioning, pagination**](./Domain_2/D2_08_REST-API-Design-Patterns-Versioning-pagination.md)
9. [**API Gateway Patterns: Routing, auth, throttling**](./Domain_2/D2_09_API-Gateway-Patterns-Routing-auth-throttling.md)
10. [**Service Mesh Concepts: Istio, Linkerd**](./Domain_2/D2_10_Service-Mesh-Concepts-Istio-Linkerd.md)
11. [**CDN Architecture and Cache Invalidation**](./Domain_2/D2_11_CDN-Architecture-and-Cache-Invalidation.md)
12. [**Load Balancing: L4 vs. L7, routing algorithms**](./Domain_2/D2_12_Load-Balancing-L4-vs-L7-routing-algorithms.md)
13. [**Reverse Proxy vs. Forward Proxy**](./Domain_2/D2_13_Reverse-Proxy-vs-Forward-Proxy.md)
14. [**P2P Networking and WebRTC**](./Domain_2/D2_14_P2P-Networking-and-WebRTC.md)
15. [**Network Partition and Split-Brain Handling**](./Domain_2/D2_15_Network-Partition-and-Split-Brain-Handling.md)


### **Domain 3: Data Storage and Databases**
1. [**Relational Database Schema Design and Indexing**](./Domain_3/D3_01_Relational-Database-Schema-Design-and-Indexing.md)
2. [**Query Optimization and Execution Plans**](./Domain_3/D3_02_Query-Optimization-and-Execution-Plans.md)
3. [**Transactions and Isolation Levels**](./Domain_3/D3_03_Transactions-and-Isolation-Levels.md)
4. [**Distributed SQL Systems**](./Domain_3/D3_04_Distributed-SQL-Systems.md)
5. [**NoSQL Categories: Document, key-value, wide-column, graph**](./Domain_3/D3_05_NoSQL-Categories-Document-key-value-wide-column-graph.md)
6. [**Document Stores and Schema Patterns**](./Domain_3/D3_06_Document-Stores-and-Schema-Patterns.md)
7. [**Key-Value Stores and TTL/Eviction Strategies**](./Domain_3/D3_07_Key-Value-Stores-and-TTL-Eviction-Strategies.md)
8. [**Wide-Column Stores: Partitioning and compaction**](./Domain_3/D3_08_Wide-Column-Stores-Partitioning-and-compaction.md)
9. [**Graph Databases and Query Models**](./Domain_3/D3_09_Graph-Databases-and-Query-Models.md)
10. [**Time-Series Database Design**](./Domain_3/D3_10_Time-Series-Database-Design.md)
11. [**Search Engines and Relevance Architecture**](./Domain_3/D3_11_Search-Engines-and-Relevance-Architecture.md)
12. [**Vector Databases and ANN Indexes**](./Domain_3/D3_12_Vector-Databases-and-ANN-Indexes.md)
13. [**NewSQL Approaches**](./Domain_3/D3_13_NewSQL-Approaches.md)
14. [**Replication Models: Sync/async, leader-follower, multi-master**](./Domain_3/D3_14_Replication-Models-Sync-async-leader-follower-multi-master.md)
15. [**Sharding Strategy and Hotspot Mitigation**](./Domain_3/D3_15_Sharding-Strategy-and-Hotspot-Mitigation.md)
16. [**Read Replicas and Query Routing**](./Domain_3/D3_16_Read-Replicas-and-Query-Routing.md)
17. [**Partitioning: Range, hash, list, composite**](./Domain_3/D3_17_Partitioning-Range-hash-list-composite.md)
18. [**Object Storage Architecture and Access Patterns**](./Domain_3/D3_18_Object-Storage-Architecture-and-Access-Patterns.md)
19. [**Data Archival and Storage Tiering**](./Domain_3/D3_19_Data-Archival-and-Storage-Tiering.md)
20. [**Zero-Downtime Database Migration**](./Domain_3/D3_20_Zero-Downtime-Database-Migration.md)
21. [**Polyglot Persistence**](./Domain_3/D3_21_Polyglot-Persistence.md)
22. [**Connection Pooling and DB Proxy Patterns**](./Domain_3/D3_22_Connection-Pooling-and-DB-Proxy-Patterns.md)

### **Domain 4: Caching**
1. **Cache Fundamentals and Hit Ratio Optimization** 
2. **Cache Placement Layers:** Client/CDN/App/DB 
3. **Caching Strategies:** Cache-aside, read-through, write-through, write-behind 
4. **Redis Patterns:** Data structures, streams, pub/sub 
5. **Distributed Cache Topology and Hashing** 
6. **Cache Stampede Mitigation** 
7. **Cache Invalidation Strategies** 
8. **Probabilistic Data Structures:** Bloom filters, HyperLogLog 
9. **Hot-Key Mitigation** 

### **Domain 5: Scalability**
1. **Vertical vs. Horizontal Scaling** 
2. **Auto-Scaling:** Reactive, predictive, scheduled 
3. **Stateless Scaling and Session Externalization** 
4. **Read Scaling via Replicas and Caching** 
5. **Write Scaling via CQRS and Event-Sourcing** 
6. **Consistent Hashing with Virtual Nodes** 
7. **Cell-Based Architecture** 
8. **Bulkhead Isolation and Blast-Radius Control** 
9. **Backpressure and Flow Control** 
10. **Data Locality and Affinity Routing** 
11. **Multi-Region and Geo-Distributed Architecture** 
12. **Elasticity vs. Scalability** 

### **Domain 6: Reliability and Availability**
1. **High Availability Patterns:** Active-active, active-passive 
2. **Failover Mechanisms and Trade-offs** 
3. **Circuit Breaker Pattern** 
4. **Retry Strategy with Exponential Backoff and Jitter** 
5. **Timeouts and Deadline Propagation** 
6. **Liveness/Readiness/Startup Checks** 
7. **Chaos Engineering Principles and Tooling** 
8. **Disaster Recovery:** RTO/RPO planning 
9. **Idempotency in Distributed Workflows** 
10. **Exactly-Once Semantics Trade-offs** 
11. **Redundancy and Replication for Resilience** 
12. **Runbooks and Game-Day Operations** 
13. **Degraded-Mode Operation Design** 

### **Domain 7: Messaging and Event-Driven Architecture**
1. **Synchronous vs. Asynchronous Communication** 
2. **Queueing Systems (Point-to-Point)** 
3. **Pub/Sub Systems and Fanout** 
4. **Kafka Architecture:** Partitions, offsets, consumer groups 
5. **Event-Driven Architecture Patterns** 
6. **Event Sourcing** 
7. **CQRS** 
8. **Event Mesh and Event Streaming Platforms** 
9. **Outbox Pattern** 
10. **Saga Pattern:** Orchestration vs. choreography 
11. **Dead-Letter Queue Strategy** 
12. **Stream vs. Batch Processing** 
13. **Schema Evolution and Compatibility** 
14. **Delivery Guarantees and Exactly-Once Processing** 

### **Domain 8: Microservices and Service Architecture**
1. **Monolith vs. Microservices Decision Framework** 
2. **Service Decomposition Strategies** 
3. **Inter-Service Communication Patterns** 
4. **API Gateway and Backend-for-Frontend (BFF)** 
5. **Service Discovery Patterns** 
6. **Sidecar and Ambassador Patterns** 
7. **Distributed Tracing in Service Ecosystems** 
8. **Modular Monolith Architecture** 
9. **Strangler Fig Migration Pattern** 
10. **Service Versioning and Compatibility** 
11. **Contract Testing** 
12. **Micro-Frontend Architecture** 
13. **Shared Library Governance in Distributed Teams** 

### **Domain 9: Security in System Design**
1. **Authentication Patterns and Credential Lifecycle** 
2. **OAuth 2.0 and OpenID Connect** 
3. **Authorization Models:** RBAC, ABAC, ReBAC 
4. **Enterprise SSO with SAML** 
5. **Zero Trust Architecture** 
6. **Encryption in Transit and at Rest** 
7. **Secrets Management Patterns** 
8. **API Security Hardening** 
9. **DDoS Mitigation Architecture** 
10. **Software Supply-Chain Security (SBOM, Signing)** 
11. **Data Privacy and Compliance Architecture** 
12. **Audit Logging and Tamper Evidence** 
13. **Threat Modeling (STRIDE, PASTA)** 
14. **WAF and IDS/IPS Integration** 

### **Domain 10: Distributed Systems Theory**
1. **Consensus Algorithms:** Paxos and Raft 
2. **Leader Election Strategies** 
3. **Distributed Locks and Coordination Services** 
4. **Logical Clocks and Causality** 
5. **Lamport Timestamps** 
6. **Byzantine Fault Tolerance** 
7. **Two-Phase and Three-Phase Commit** 
8. **Distributed Transaction Models** 
9. **CRDTs for Conflict-Free Replication** 
10. **Gossip Protocols** 
11. **Quorum-Based Read/Write Systems** 
12. **Linearizability vs. Serializability** 
13. **Fallacies of Distributed Computing** 

### **Domain 11: Cloud and Infrastructure**
1. **Cloud Service Models:** IaaS, PaaS, SaaS, FaaS 
2. **Shared Responsibility Model** 
3. **VPC/Subnet/Routing/Security Architecture** 
4. **Multi-Cloud and Hybrid Patterns** 
5. **Infrastructure as Code (IaC)** 
6. **Serverless Architecture and Trade-offs** 
7. **Container Fundamentals** 
8. **Kubernetes Architecture and Workloads** 
9. **Kubernetes Advanced Patterns:** Operators, CRDs 
10. **Service Mesh on Kubernetes** 
11. **Cloud Cost Optimization Design** 
12. **Landing Zone Architecture** 
13. **FinOps Operating Model** 
14. **Spot/Preemptible Instance Strategies** 

### **Domain 12: Data Engineering and Analytics**
1. **Data Warehouse vs. Data Lake vs. Lakehouse** 
2. **ETL vs. ELT** 
3. **Batch Processing Architecture** 
4. **Stream Processing Architecture** 
5. **Lambda Architecture** 
6. **Kappa Architecture** 
7. **Data Mesh Principles** 
8. **Metadata, Catalog, and Governance** 
9. **CDC (Change Data Capture) Patterns** 
10. **Data Quality and Lineage** 
11. **Schema Registry and Data Contracts** 
12. **Columnar Formats:** Parquet, ORC, Arrow 
13. **OLAP Engines and Analytical Serving** 
14. **Feature Stores for ML Systems** 

### **Domain 13: Observability and SRE**
1. **Logs, Metrics, and Traces Fundamentals** 
2. **Structured Logging Architecture** 
3. **Metrics Collection and Dashboard Strategy** 
4. **Distributed Tracing and Context Propagation** 
5. **Alerting Strategy and On-Call Design** 
6. **OpenTelemetry Deep Dive** 
7. **SRE:** Error budgets and toil reduction 
8. **Synthetic Monitoring and Canary Checks** 
9. **Continuous Profiling** 
10. **AIOps for Anomaly Detection and Auto-remediation** 

### **Domain 14: CI/CD and Delivery**
1. **Deployment Strategies:** Rolling, canary, blue-green, shadow 
2. **CI/CD Pipeline Architecture** 
3. **Feature Flag Architecture** 
4. **Branching Strategies:** Trunk-based vs. Gitflow 
5. **GitOps Delivery Model** 
6. **Supply Chain Controls in CI/CD** 
7. **Test Strategy Portfolio** 
8. **Database Migration in Release Pipelines** 
9. **Artifact Versioning and Promotion** 

### **Domain 15: Real-Time and Streaming Systems**
1. **Real-time vs. Near-real-time Trade-offs** 
2. **Chat System Architecture** 
3. **Notification Architecture:** Multi-channel fanout 
4. **Presence and Typing Indicators at Scale** 
5. **Live Video Streaming Design** 
6. **Collaborative Editing (OT vs. CRDT)** 
7. **Geospatial / Ride-Hailing Systems** 
8. **Low-Latency Trading System Design** 
9. **IoT Ingestion and Edge Processing** 

### **Domain 16: AI and ML Systems**
1. **ML Lifecycle Architecture:** Train, evaluate, serve 
2. **Inference Patterns:** Online, batch, stream 
3. **Feature Stores and Consistency** 
4. **MLOps:** Model registry, lineage, drift monitoring 
5. **Experimentation and A/B Testing Platforms** 
6. **Shadow Deployments and Canaries for ML** 
7. **LLM Inference Infrastructure** 
8. **Retrieval-Augmented Generation (RAG) Architecture** 
9. **Agentic System Orchestration and Tool Use** 
10. **Vector Search at Scale** 
11. **Responsible AI Controls** 
12. **Multi-modal System Architecture** 

### **Domain 17: API and Developer Experience**
1. **REST Resource Modeling and HTTP Semantics** 
2. **API Versioning Lifecycle** 
3. **Pagination Patterns:** Cursor vs. offset 
4. **GraphQL Federation** 
5. **Webhook Platform Design** 
6. **SDK Design and Dev Portal Architecture** 
7. **OpenAPI/AsyncAPI Contract-First Design** 
8. **Rate Limiting Algorithms** 
9. **API Monetization and Quota Systems** 
10. **API Governance:** Internal vs. external 

### **Domain 18: Frontend and Mobile Architecture**
1. **SPA vs. SSR vs. SSG vs. ISR** 
2. **Micro-Frontend Patterns** 
3. **State Management Strategies** 
4. **PWA and Offline-First Architecture** 
5. **Web Performance Architecture (Core Web Vitals)** 
6. **Mobile Architecture Patterns** 
7. **Cross-Platform vs. Native Trade-offs** 
8. **Offline Sync and Conflict Resolution** 
9. **Frontend Observability and RUM** 

### **Domain 19: Enterprise Architecture Patterns**
1. **Enterprise Integration Patterns** 
2. **Domain-Driven Design (DDD)** 
3. **Hexagonal Architecture** 
4. **Clean Architecture** 
5. **TOGAF Overview and Adaptation** 
6. **Architecture Decision Records (ADRs)** 
7. **Architecture Fitness Functions** 
8. **Platform Engineering and Internal Developer Platforms (IDPs)** 
9. **Technical Debt Management Frameworks** 
10. **API-First Enterprise Strategy** 
11. **Team Topologies and Conway’s Law** 
12. **Evolutionary Architecture** 

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
