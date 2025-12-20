Operational Hardening: Architectural Addendum v2.0
1. Runtime Execution Environment
To satisfy the P95 < 800ms latency requirement while running CPU-intensive Natural Language Inference (NLI) workloads, the execution environment deviates from standard serverless defaults.

Node.js Concurrency Tuning
Standard serverless concurrency (often 80 requests/instance) is incompatible with the single-threaded Node.js event loop during regex-heavy parsing or tokenization.

Hard Limit: Container concurrency is strictly capped at 10 concurrent requests.

Scale-Out Behavior: The scheduler prioritizes horizontal scaling (adding instances) over vertical queuing to prevent event loop blocking.

Cold Start Mitigation: min-instances: 1 and Startup CPU Boost are mandated for the API Gateway to guarantee P50 < 200ms.

2. Asynchronous Logic & The "202" Contract
The system explicitly separates Verification Logic from Response Delivery for deep semantic analysis.

The "202 Accepted" Pattern: If Semantic Entailment (Gate B) requires > 2,000ms (e.g., fetching large PDFs, running large NLI models), the API immediately returns HTTP 202 Accepted. The client must handle the eventual consistency of the score via webhook or polling.

Isolation of Concerns:

Hot Path (Sync): Availability checks, Metadata scoring, Cache hits.

Cold Path (Async): NLI Inference, Scraping, Audit Logging.

Transport: Inter-service communication relies on Cloud Pub/Sub to decouple the API response from the durability requirements of the Ancestry Log.

3. Database Resilience Strategy
Given the high elasticity of the serverless compute layer, direct database connections are prohibited to prevent exhaustion attacks.

Connection Pooling: All database interactions must route through a PgBouncer sidecar configured in Transaction Mode. Application-level pool size is fixed at max: 1 per container.

Migration Integrity: Schema migrations must utilize Cryptographic Checksums (row-hash verification) rather than relying solely on exit codes. This ensures zero data corruption during schema evolution.

4. Automated Safety Protocols
Deployment pipelines implement a "Dead Man's Switch" based on real-time telemetry.

Traffic Splitting: New revisions deploy to 0% traffic initially, graduating to 100% only after passing synthetic health checks.

Rollback Triggers: An automated rollback to the stable revision is triggered immediately if:

Error Rate (HTTP 5xx) > 1% over a 1-minute window.

P95 Latency > 2,000ms over a 5-minute window.
