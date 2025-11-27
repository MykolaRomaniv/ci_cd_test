## **1\. System Architecture & Diagram**

### **Main components**

* **Frontend**: React SPA  
  * Built as static assets.  
  * Hosted in **Cloud Storage**.  
  * Fronted by **Cloud CDN** \+ **External HTTPS Load Balancer**.

* **Backend**: FastAPI  
  * Containerized and deployed to **Cloud Run (fully managed)**.  
  * Exposed via the same **HTTPS Load Balancer** under /api.

* **Database**: MongoDB  
  * **Managed MongoDB Atlas cluster on GCP** (preferred for simplicity & reliability).  
  * Deployed in the same region as Cloud Run.  
  * Connected via **VPC peering** between GCP VPC and Atlas VPC.

* **Supporting services**  
  * **Artifact Registry** for container images.  
  * **Cloud Build** for CI/CD.  
  * **Secret Manager** for secrets.  
  * **Cloud Monitoring & Logging** for observability.  
  * **Cloud Armor** (optional but recommended) for WAF/rate limiting.

  ### **Networking & traffic flow**

* **VPC (per environment)**:  
  * One **custom VPC** with subnets in at least 2 zones.  
  * External HTTPS LB & Cloud CDN are public.  
  * Cloud Run is regional, attached to VPC via **Serverless VPC Connector** for private DB access.  
  * MongoDB Atlas is reachable only via **VPC peering** (no public IPs).

**Health checks**:

* HTTPS LB performs HTTP health checks on **/health** endpoint of the FastAPI service.  
* React SPA health (static hosting) is checked by LB verifying 200 on /index.html.  
* MongoDB Atlas has its own cluster health & monitoring; application also exposes DB connectivity status on /health if needed.

**Service discovery**:

* Frontend calls backend via a **single API base URL** (e.g. https://api.myapp.com) configured as build-time env variable.  
* Backend reads Mongo connection string from Secret Manager; DNS resolution to Atlas via VPC peering.

**Internal vs external traffic**:

* Only **HTTPS LB** and **Cloud CDN** are exposed to the public internet.  
* Cloud Run service is configured with **“internal \+ LB”** access (no direct public invocation).  
* MongoDB cluster has **no public endpoint**; access only from the peered VPC.

## **2\. Deployment Environments & CI/CD**

### **Environment separation**

Use **separate GCP projects** for clean isolation:

* myapp-dev  
* myapp-stg  
* myapp-prod

Each project has:

* Its own **VPC**, **Cloud Run service**, **Cloud Storage bucket**, **Secret Manager entries**, and **MongoDB database/cluster or separate DBs** within a shared Atlas project.  
* Per-env domains, e.g.:  
  * dev.myapp.com, dev-api.myapp.com  
  * stg.myapp.com, stg-api.myapp.com  
  * app.myapp.com, api.myapp.com

### **CI/CD pipeline (simple but robust)**

Use **Cloud Build** triggered from GitHub/GitLab:

1. **On push / PR to develop** (dev):  
   * Run unit tests (frontend & backend).  
   * Build Docker image for FastAPI → push to **Artifact Registry**.  
   * Build React → upload static assets to **dev** Cloud Storage bucket.  
   * Deploy new revision of FastAPI to **Cloud Run (dev)**.  
   * Optionally, run smoke tests.

2. **On merge to main** (staging):  
   * Same steps as dev but targeting **staging** project/resources.  
   * Deploy using **canary rollout** (e.g. 10% traffic to new revision).  
   * Run integration tests against staging.

3. **Promotion to production**:  
   * Triggered by **manual approval** in Cloud Build.  
   * Reuse **the same images/artifacts** that passed in staging (immutable artifact promotion).  
   * Deploy to **Cloud Run (prod)** with canary (see zero-downtime section).  
   * Sync React build to prod bucket.  
   * Post-deployment smoke tests and rollback if needed.

This keeps CI/CD straightforward: **one pipeline**, with **per-env config** controlled via substitution variables (project ID, bucket name, Cloud Run service name, Secret Manager paths).

## **3\. Zero-Downtime Deployment Strategy – Canary on Cloud Run**

**I’d use canary releases leveraging Cloud Run’s traffic splitting by revision:**

* **For each new backend release:**

1. Deploy a new Cloud Run revision (no traffic yet).  
2. Shift 5–10% of traffic to the new revision.  
3. Monitor key metrics (5xx rate, latency, error logs) for a defined window (e.g. 30–60 minutes).  
4. If healthy, gradually increase to 50% and then 100%.  
5. If issues arise, instantly rollback by routing 100% traffic back to the previous revision (no redeploy needed).

**When/why:**

* **Use canary for all production deployments, especially when:**  
  * Changes are non-trivial (DB queries, auth, business logic).  
  * We want to protect SLOs while still continuously delivering.

**For the frontend, because it’s static:**

* Upload new build to a versioned path in Cloud Storage (e.g. /releases/2025-11-27/).  
* Update load balancer backend or rewrite rule to point to the new version.  
* If something breaks, switch back to the previous version (effectively a blue/green pattern for static assets).

## **4\. Scalability & High Availability**

### **Horizontal scaling**

* **Cloud Run**:  
  * Auto-scales based on **concurrent requests**.  
  * Configure:  
    * max-instances high enough to handle expected peak.  
    * concurrency (e.g. 40–80) depending on CPU intensity of FastAPI endpoints.

* **React SPA**:  
  * Cloud Storage \+ Cloud CDN are inherently horizontally scalable; no app server to scale.

* **MongoDB Atlas**:  
  * Use a **3-node replica set** spread across **multiple zones** in the same region.  
  * Vertically scale cluster tier as necessary.  
  * Optionally enable auto-scaling (Atlas feature).

### **CPU/memory auto-scaling**

* Cloud Run:  
  * Configure **CPU/memory per instance** based on profiling (e.g. 1 vCPU, 512–1024MB).  
  * Requests drive instance count; no need to manually handle CPU scaling.

### **Load balancing & multi-zone HA**

* **External HTTPS Load Balancer**:  
  * Global LB that routes traffic to Cloud CDN / Cloud Storage and Cloud Run.  
  * Regionally deploy Cloud Run in a region with **multiple zones**.

* **High availability**:  
  * Cloud Run is **regional multi-zonal**.  
  * MongoDB replica set ensures failover between zones.  
  * Buckets & CDN are replicated; no single-zone dependency for static content.

## **5\. Scalability & High Availability**

### **Horizontal scaling**

* **Cloud Run**:  
  * Auto-scales based on **concurrent requests**.  
  * Configure:  
    * max-instances high enough to handle expected peak.  
    * concurrency (e.g. 40–80) depending on CPU intensity of FastAPI endpoints.

* **React SPA**:  
  * Cloud Storage \+ Cloud CDN are inherently horizontally scalable; no app server to scale.

* **MongoDB Atlas**:  
  * Use a **3-node replica set** spread across **multiple zones** in the same region.  
  * Vertically scale cluster tier as necessary.  
  * Optionally enable auto-scaling (Atlas feature).

### **CPU/memory auto-scaling**

* Cloud Run:  
  * Configure **CPU/memory per instance** based on profiling (e.g. 1 vCPU, 512–1024MB).  
  * Requests drive instance count; no need to manually handle CPU scaling.

### **Load balancing & multi-zone HA**

* **External HTTPS Load Balancer**:  
  * Global LB that routes traffic to Cloud CDN / Cloud Storage and Cloud Run.  
  * Regionally deploy Cloud Run in a region with **multiple zones**.

* **High availability**:  
  * Cloud Run is **regional multi-zonal**.  
  * MongoDB replica set ensures failover between zones.  
  * Buckets & CDN are replicated; no single-zone dependency for static content.

## **6\. Monitoring & Alerting**

Use **Cloud Monitoring** \+ **Cloud Logging**.

### **Metrics to collect**

* **Backend (Cloud Run / FastAPI)**:  
  * Request count, latency (p50, p95, p99).  
  * HTTP 4xx and 5xx rates.  
  * Instance CPU & memory utilization.  
  * Concurrency and instance count.

* **Frontend**:  
  * LB & Cloud CDN metrics (request count, cache hit ratio).  
  * Optional: Web Vitals and frontend logs sent to backend.

* **Database (MongoDB)**:  
  * From Atlas: operations per second, CPU/IO, replication lag, slow queries.

### **Logs**

* Application logs from FastAPI via **structured JSON logging** to Cloud Logging:  
  * Request traces (request ID, user ID if available).  
  * Error stack traces.  
  * Significant business events.  
* LB and CDN access logs.  
* MongoDB Atlas logs (in Atlas console; optionally exported to a centralized logging solution if needed).

### **Dashboards**

In **Cloud Monitoring**:

1. **API Service Dashboard**:  
   * Requests, latency, error rate by endpoint.  
   * Instance count, CPU/memory per Cloud Run service.

2. **Frontend / User Experience Dashboard**:  
   * CDN/LB traffic.  
   * Cache hit ratio, response times.

3. **DB Health Dashboard** (via Atlas UI):  
   * Ops/sec, slow query count, CPU/IO, replication lag.

### **Alerts**

Configure alerts for:

* **5xx error rate** above threshold (e.g. \>1% over 5–10 minutes).  
* **Latency spikes** (p95 \> X ms for sustained period).  
* **Cloud Run instance failures** or abnormal restarts.  
* **DB metrics**:  
  * High replication lag.  
  * Disk / CPU saturation.  
* **Error budget burn rate** (if SLOs are defined).

Alerts can be sent to **Slack, email, or PagerDuty**.

## **7\. Security & Hardening**

### **Secrets handling**

* All sensitive data in **Secret Manager**, never in source code or plain env files.  
* Cloud Run service account granted only secretmanager.versions.access for specific secrets.

### **IAM roles & permissions**

* **Least privilege**:  
  * Separate service accounts:  
    * frontend-build-sa (Cloud Build).  
    * backend-run-sa (Cloud Run runtime).  
  * Backend SA only has:  
    * Access to Secret Manager secrets.  
    * Access to VPC connector.  
  * IAM for developers is scoped per environment (e.g. dev can deploy, prod deployment requires approval).

### **HTTPS/TLS**

* External access only via **HTTPS**:  
  * HTTPS Load Balancer terminates TLS using **managed certificates**.  
* If needed, mTLS between backend and DB can be configured via Atlas.

### **Rate limiting & WAF**

* Use **Cloud Armor** on the HTTPS LB for:  
  * Basic rate limiting per IP.  
  * Protection against common OWASP top 10 attacks.  
  * Geo-based rules if necessary.

### **Firewalls / security groups**

* **VPC firewall rules**:  
  * Allow Cloud Run to access MongoDB Atlas IP ranges via VPC peering.  
  * Deny all inbound traffic to internal subnets from the internet.  
* MongoDB: no public IP; only peered VPC access.

### **Audit logging**

* Enable **Cloud Audit Logs** for:  
  * Admin operations (deployments, IAM changes).  
  * Access to Secret Manager.  
* Periodically review logs and set alerts for suspicious activities (e.g. repeated secret access failures).

## **8\. Backup & Recovery**

### **DB backup strategy**

* Rely on **MongoDB Atlas automated backups**:  
  * Daily snapshots \+ **point-in-time recovery**.  
  * Retention set according to requirements (e.g. 14–30 days).  
* Test recovery regularly into a **separate staging cluster** to verify backup integrity.

### **RTO & RPO**

* **RPO (Recovery Point Objective)**:  
  * With point-in-time backups, RPO can be as low as **a few minutes**.

* **RTO (Recovery Time Objective)**:  
  * Target **\< 1 hour** for major DB failure:  
    * Spin up a new Atlas cluster from backups.  
    * Update DB connection string in Secret Manager.  
    * Redeploy/roll Cloud Run with new secret (or trigger configuration rollout).

### **Failure scenarios & response**

1. **Single instance / revision issue (bug in new release)**:  
   * Detect via canary metrics.  
   * **Rollback** by routing 100% traffic to previous Cloud Run revision.

2. **Region-level issue for Cloud Run**:  
   * Optionally maintain a **warm standby** Cloud Run service in another region.  
   * DNS or LB failover to the standby region.

3. **DB corruption / accidental data deletion**:  
   * Use Atlas **point-in-time restore** to a new cluster.  
   * Temporarily put application into **read-only mode** or maintenance.

4. **Static assets issue**:  
   * Revert LB backend to previous Cloud Storage release.  
   * One command / config change; minimal RTO.

This design uses **Cloud Storage \+ CDN, Cloud Run, Secret Manager, Cloud Build, Artifact Registry, and MongoDB Atlas** as the core building blocks. It deliberately avoids Kubernetes and other complex components, keeping the system **operationally simple** while still supporting **zero-downtime deployments, horizontal scaling, security best practices, and robust backup & recovery**.