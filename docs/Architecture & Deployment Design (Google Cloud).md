## **PART 1 — Architecture & Deployment Design (Google Cloud)**

### **1\. High-Level Overview**

We have a 3-service system:

* **React frontend** (SPA)

* **FastAPI backend** (REST API)

* **MongoDB database**

Target: production-ready deployment on **Google Cloud Platform (GCP)** with CI/CD, zero-downtime releases, observability, and separate Dev / Staging / Production environments.

---

### **2\. System Architecture**

**Main components (GCP):**

* **Client:** Browser.

* **Static hosting \+ CDN:** React build in **Cloud Storage** bucket, fronted by **Cloud CDN** via an external HTTPS Load Balancer.

* **API entrypoint:** **External HTTP(S) Load Balancer** routing traffic to **GKE** (Google Kubernetes Engine) services.

* **Backend:** FastAPI containers running as Deployments in **GKE** (private nodes).

* **Database:** **MongoDB Atlas on GCP** (or self-managed MongoDB on GCE/GKE) connected via **VPC peering / Private Service Connect**.

* **Networking:**

  * Single **VPC**.

  * **Public subnets** for load balancer frontend (and optionally bastion).

  * **Private subnets** for GKE node pools and MongoDB.

  * **Cloud NAT** for outbound internet access from private workloads.

* **Service discovery:** Kubernetes Services \+ DNS; health checks via Load Balancer and K8s probes.

**Traffic flow:**

* Browser → HTTPS Load Balancer \+ Cloud CDN → Cloud Storage (static assets).

* Browser/API calls → HTTPS Load Balancer → GKE (FastAPI service).

* FastAPI → MongoDB Atlas over private network.

### **3\. Deployment Environments**

**Environments:**

* **Dev**

  * Single **GKE** cluster with `dev` namespace.

  * Small node pool (preemptible nodes optional to save cost).

  * Dev MongoDB Atlas project/cluster on lower tier.

  * Used for feature branches and quick feedback.

* **Staging**

  * Separate `staging` namespace (or separate GKE cluster if required).

  * Mirrors production config as closely as possible.

  * Uses anonymized/synthetic data.

  * Target for automated e2e tests and release validation.

* **Production**

  * Dedicated **GKE** cluster (separate project or at least separate VPC).

  * **Multi-zone** node pools for HA.

  * Production MongoDB Atlas cluster with replica set across zones.

  * Strict IAM, network boundaries, and higher resource limits.

**Promotion flow:**

1. **Dev:** Every push/PR to `dev` → build, test, deploy to dev namespace.

2. **Staging:** Merge to `main` or release branch → deploy to staging.

3. **Production:** Only tagged releases (`vX.Y.Z`) and **manual approval** (e.g., protected environment / Cloud Deploy approval) → deploy to prod using same container images.

This keeps artefacts **immutable** across environments.

### **5\. Zero-Downtime Deployment**

Use **GKE Deployments** with **rolling updates**:

* K8s Deployment configured with:

  * `maxSurge: 1`

  * `maxUnavailable: 0`

* **Readiness probes** gate traffic until containers are healthy.

* **Liveness probes** ensure unhealthy pods are restarted.

Result: old pods stay serving until new pods are up and ready → **no downtime**.

For larger scale / higher-risk changes:

* **Blue/Green (on GKE):** Two versions of the deployment or two Services/Ingresses; switch backend service on the HTTP(S) Load Balancer.

* **Canary:** Separate Service/Deployment and route a percentage of traffic via **Traffic Director** or custom ingress rules.

For this app, state: **Rolling updates as default**, Blue/Green or Canary for big risky releases.

### **6\. Scalability Plan**

**Horizontal scaling:**

* Backend:

  * GKE **Horizontal Pod Autoscaler (HPA)** based on CPU/requests/latency.

  * Multiple replicas per zone for HA.

* Frontend:

  * Static content via Cloud Storage \+ Cloud CDN scales automatically.

* DB:

  * MongoDB Atlas cluster with auto-scaling for CPU, storage, and connections.

**Vertical scaling:**

* Resize GKE node pool machine types if CPU/memory consistently high.

* Scale MongoDB Atlas cluster tier.

**High availability:**

* **Multi-zone** GKE cluster (e.g., `europe-west3-a/b/c`).

* MongoDB replica set across zones in the same region.

* External Load Balancer across zones for resilient ingress.

### **7\. Monitoring & Alerting**

Use **Cloud Monitoring** \+ **Cloud Logging** as the primary observability stack.

**Metrics to track:**

* **FastAPI / backend:**

  * Request rate, latency (p95/p99), 4xx/5xx error rates.

  * Pod restarts, HPA scaling events.

* **MongoDB:**

  * CPU, memory, connections, replication lag, slow queries (via Atlas).

* **Infra:**

  * GKE node CPU/RAM, disk, network.

  * Load Balancer request/latency metrics.

Tooling:

* **Cloud Monitoring** dashboards:

  * API performance dashboard.

  * GKE cluster and node health.

* **Cloud Logging:**

  * Structured JSON logs from FastAPI, tagged with trace IDs/request IDs.

* Optionally integrate:

  * **Cloud Trace** and **Cloud Profiler** for deeper performance insight.

  * Atlas monitoring for DB-level visibility.

**Alerts (Cloud Monitoring Alerting Policies):**

* 5xx error rate \> N% for X minutes.

* Latency p95 above threshold.

* GKE node pool CPU \>80% for sustained period.

* MongoDB Atlas CPU / connections / disk near limits.

* Frequent pod restarts or failing health checks.

### **8\. Security & Hardening**

* **Secrets:**

  * Store secrets in **Secret Manager**.

  * Access via GKE Workload Identity and service accounts.

* **IAM:**

  * Principle of least privilege for:

    * GKE node service accounts.

    * CI/CD service accounts (build and deploy).

    * Application service accounts (read-only access to specific secrets, logs, etc.).

* **Network security:**

  * VPC with private subnets for GKE and DB.

  * **Cloud NAT** for outbound internet from private workloads.

  * **VPC peering / Private Service Connect** with MongoDB Atlas; DB not exposed publicly.

  * **GKE Network Policies** to restrict pod-to-pod communication if needed.

* **TLS / HTTPS:**

  * **Managed SSL certificates** on the External HTTPS Load Balancer.

  * All client traffic encrypted; internal service traffic restricted to private network.

* **WAF & rate limiting:**

  * **Cloud Armor** policies in front of Load Balancer for:

    * Basic OWASP protections.

    * IP allow/deny lists.

    * Rate limiting rules.

  * Optional app-level throttling on sensitive endpoints.

* **Audit & access logging:**

  * Load Balancer access logs to Cloud Logging.

  * Admin / auth actions logged in backend.

  * **Cloud Audit Logs** for service-level operations.

### **9\. Backup & Recovery**

* **Database backups:**

  * Use **MongoDB Atlas automated backups** with point-in-time recovery (PITR) for production.

  * Retention policy (e.g., 7–30 days depending on requirements).

* **Configuration backups:**

  * GKE manifests stored in git.

  * Optionally export cluster configs (e.g., `gcloud` \+ Terraform if used).

**RTO/RPO assumptions:**

* Example targets:

  * **RTO:** 30–60 minutes (restore DB \+ redeploy services).

  * **RPO:** \< 15 minutes (with PITR) or daily backups (if cheaper, but less strict).

* Clarify chosen values and why they’re acceptable for this application.

**Failure scenarios:**

* **Pod crash or bad deployment:**

  * K8s restarts pods automatically.

  * Roll back deployment via `kubectl rollout undo` or CI/CD tooling.

* **Node failure:**

  * GKE reschedules pods on healthy nodes in other zones.

* **MongoDB primary failure:**

  * Atlas automatic failover to a secondary node.

  * Application reconnects transparently (driver \+ retry logic).

* **Regional outage (optional extension):**

  * Secondary GCP region with a warm standby GKE cluster and replicated Atlas cluster.

  * DNS / Load Balancer failover plan documented.

