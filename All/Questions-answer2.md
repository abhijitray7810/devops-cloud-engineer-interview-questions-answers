# 🔥 DevOps Interview Trap Questions — Complete Answer Guide

> **Pro Tip:** These questions test your ability to think beyond surface-level metrics. The core skill being evaluated is **systematic debugging under ambiguity** — not just tool knowledge.

---

## Table of Contents

1. [Kubernetes pods Running, but app is down](#1-kubernetes-pods-running-but-app-is-down)
2. [Jenkins pipeline succeeded, but deployment failed](#2-jenkins-pipeline-succeeded-but-deployment-failed)
3. [Docker container healthy, but users can't access app](#3-docker-container-healthy-but-users-cant-access-app)
4. [Monitoring green, but customers report failures](#4-monitoring-green-but-customers-report-failures)
5. [Terraform apply successful, but infra partially broken](#5-terraform-apply-successful-but-infra-partially-broken)
6. [Works in staging, fails in production](#6-works-in-staging-fails-in-production)
7. [CPU normal, but response time very high](#7-cpu-normal-but-response-time-very-high)
8. [Auto Scaling triggered, but performance still poor](#8-auto-scaling-triggered-but-performance-still-poor)
9. [Rollback completed, but issue continues](#9-rollback-completed-but-issue-continues)
10. [DNS working internally, not externally](#10-dns-working-internally-not-externally)
11. [SSL certificate valid, but HTTPS failing](#11-ssl-certificate-valid-but-https-failing)
12. [Kubernetes Ingress configured, but traffic blocked](#12-kubernetes-ingress-configured-but-traffic-blocked)
13. [Logs available, but root cause unclear](#13-logs-available-but-root-cause-unclear)
14. [CI/CD fast, but deployments unstable](#14-cicd-fast-but-deployments-unstable)
15. [Terraform state correct, but cloud resources drifted](#15-terraform-state-correct-but-cloud-resources-drifted)
16. [Redis cache healthy, but app still slow](#16-redis-cache-healthy-but-app-still-slow)
17. [Kafka brokers healthy, but consumer lag increasing](#17-kafka-brokers-healthy-but-consumer-lag-increasing)
18. [Nginx healthy, but backend services overloaded](#18-nginx-healthy-but-backend-services-overloaded)
19. [Alerts triggered late during outage](#19-alerts-triggered-late-during-outage)
20. ["Everything looks healthy" but production is failing](#20-everything-looks-healthy-but-production-is-failing)

---

## 1. Kubernetes Pods Running, but App is Down

**What it indicates:** `Running` is a **process state**, not an **application state**. The container process is alive, but the application inside may be broken.

### Root Causes to Investigate

| Layer | What to Check |
|-------|---------------|
| **App startup** | Is the app still initializing? Crash-looping silently? |
| **Readiness Probe** | Pod `Running` but not `Ready` — traffic won't be routed |
| **Liveness Probe** | Not configured or misconfigured — app hangs undetected |
| **Port mismatch** | App listening on `8080`, probe/service pointing to `80` |
| **Config/Secrets** | Missing env vars, wrong DB credentials, feature flags |
| **OOMKilled** | App killed by OOM but container restarts — check `kubectl describe pod` |

### Diagnostic Commands

```bash
# Check pod readiness status
kubectl get pods -o wide

# Look at events and probe failures
kubectl describe pod <pod-name>

# Stream application logs
kubectl logs <pod-name> --follow

# Check previous crash logs
kubectl logs <pod-name> --previous

# Exec into the pod and test locally
kubectl exec -it <pod-name> -- curl localhost:8080/health
```

### Key Insight
> **Running ≠ Ready ≠ Healthy.** Always verify that readiness and liveness probes are correctly configured and passing.

---

## 2. Jenkins Pipeline Succeeded, but Deployment Failed

**How is that possible?** The pipeline reports success based on **exit codes and stage logic** — not actual deployment health.

### Root Causes to Investigate

- **Pipeline doesn't verify deployment:** `kubectl apply` exits `0` even if pods fail to start
- **Decoupled deployment step:** Jenkins triggers a deploy script but doesn't wait for rollout
- **Wrong environment targeted:** Pipeline deployed to staging, not production
- **Image pull errors:** The image tag was pushed but from a different registry or with wrong credentials
- **Post-deploy health check missing:** No step validates the app is actually up
- **Race condition:** Pipeline marked success before pods crashed after startup

### How to Fix

```groovy
// In Jenkinsfile — always wait for rollout
stage('Deploy') {
    steps {
        sh 'kubectl apply -f deployment.yaml'
        sh 'kubectl rollout status deployment/my-app --timeout=120s'
    }
}

stage('Smoke Test') {
    steps {
        sh 'curl -f https://myapp.example.com/health || exit 1'
    }
}
```

### Key Insight
> **CI success ≠ deployment success.** Always add a `rollout status` check and a post-deploy smoke test as mandatory pipeline stages.

---

## 3. Docker Container Healthy, but Users Can't Access Application

**Why?** The `HEALTHCHECK` in Docker only tests what you tell it to — often just `curl localhost`. Users hit the service from outside.

### Root Causes to Investigate

| Layer | Issue |
|-------|-------|
| **Port binding** | Container exposes `8080` but host maps to `9090` |
| **Network** | Container on a custom bridge network, not reachable externally |
| **Firewall / Security Group** | Host or cloud firewall blocks the port |
| **Reverse proxy misconfiguration** | Nginx/HAProxy in front not routing correctly |
| **Healthcheck mismatch** | `HEALTHCHECK` passes on internal port but app fails on external route |
| **TLS termination** | App expects plain HTTP but receives HTTPS at load balancer |

### Diagnostic Steps

```bash
# Verify port mapping
docker ps --format "table {{.Names}}\t{{.Ports}}"

# Test from inside vs outside
docker exec -it <container> curl localhost:8080
curl http://<host-ip>:<mapped-port>

# Check host-level firewall
iptables -L -n | grep <port>
ufw status

# Inspect network
docker network inspect <network-name>
```

### Key Insight
> **Healthy = alive inside its own world.** Reachability requires the full network path: container → Docker network → host port → firewall → load balancer → DNS → user.

---

## 4. Monitoring Dashboards Show Green, but Customers Report Failures

**What next?** Your monitoring is measuring the **wrong things** or has **blind spots**.

### Root Causes to Investigate

- **Synthetic vs real user traffic:** Dashboards use internal health-check pings, not real user paths
- **Partial failure:** Only a subset of users affected (specific region, browser, ISP, or feature)
- **Wrong metrics tracked:** CPU/memory green but error rate for specific endpoints is spiking
- **Missing business metrics:** No tracking of failed checkouts, login errors, or specific API failures
- **Alert thresholds too loose:** Errors below threshold don't trigger alerts
- **Third-party dependency failure:** Payment gateway, CDN, or external API failing — not your infra

### Immediate Response Checklist

```
1. [ ] Check real user monitoring (RUM) — not just synthetic checks
2. [ ] Query error rates by endpoint, not just overall
3. [ ] Check third-party service status pages
4. [ ] Segment by geography, device, user segment
5. [ ] Correlate with recent deployments or config changes
6. [ ] Review application logs — not just infrastructure metrics
```

### Key Insight
> **Green dashboards are only as good as the questions they answer.** Instrument user journeys, not just server health.

---

## 5. Terraform Apply Successful, but Infrastructure Partially Broken

**Why?** `terraform apply` succeeds as long as the API calls return success — it doesn't validate that resources are **functionally operational**.

### Root Causes to Investigate

- **Eventual consistency:** Cloud resources provisioned but not yet fully propagated (IAM, DNS, etc.)
- **Dependency ordering:** Resource B created before Resource A is fully ready
- **Soft limits / quota errors:** Resource created but capped or throttled silently
- **Provider bugs:** Terraform provider returns success but the cloud API had a partial failure
- **Data sources stale:** `data` blocks cached old values from previous state
- **Manual changes outside Terraform:** Previously drifted resources conflict with new config

### Diagnostic Steps

```bash
# Check what Terraform thinks exists
terraform show

# Detect drift between state and reality
terraform plan

# Refresh state from actual cloud
terraform refresh

# Validate individual resources in AWS/GCP/Azure console
# Cross-reference with cloud provider audit logs
```

### Key Insight
> **IaC success = API accepted the request.** Not that the infrastructure works end-to-end. Always validate with integration tests after `terraform apply`.

---

## 6. Application Works in Staging but Fails in Production

**What could be different?** Almost everything. Staging parity is a myth unless deliberately enforced.

### Common Differences

| Category | Staging | Production |
|----------|---------|------------|
| **Scale** | 1 replica | 10+ replicas — race conditions surface |
| **Data volume** | Small seed data | Millions of records — slow queries |
| **Secrets/Config** | Test API keys | Real keys with different permissions |
| **Network** | Open/permissive | Strict firewall, VPC peering, private endpoints |
| **Feature Flags** | All on | Gradual rollout — inconsistent states |
| **External services** | Mocked or sandbox | Real third-party APIs with rate limits |
| **TLS/SSL** | Self-signed or HTTP | Valid certs with strict HSTS |
| **Caching** | Empty cache | Warmed cache with stale data |

### How to Narrow It Down

```bash
# Compare environment variables
diff <(kubectl get configmap -n staging -o yaml) <(kubectl get configmap -n prod -o yaml)

# Check resource limits
kubectl describe pod <pod> -n prod | grep -A5 Limits

# Reproduce with production data volume in staging
# Enable verbose/debug logging temporarily in production
```

### Key Insight
> **Staging-production parity is a discipline, not a default.** Use feature flags, canary deployments, and infrastructure-as-code to minimize environment divergence.

---

## 7. CPU Usage Normal, but Application Response Time Very High

**Why?** Latency is not caused by CPU alone. Many bottlenecks are **I/O-bound**, not compute-bound.

### Root Causes to Investigate

- **Database:** Slow queries, missing indexes, lock contention, connection pool exhausted
- **Network I/O:** High latency to downstream services, DNS resolution delays
- **Memory pressure:** Garbage collection pauses (JVM, Go, Python GC)
- **Thread starvation:** All threads waiting on I/O — CPU idle but requests queued
- **External API calls:** Synchronous calls to slow third-party services blocking threads
- **Disk I/O:** Logging too verbosely, temp file writes, full disk
- **Connection pool limits:** DB pool saturated — requests wait even though CPU is free

### Diagnostic Tools

```bash
# Application-level tracing (APM)
# Check distributed trace waterfalls in Datadog / Jaeger / New Relic

# Database slow query log
SHOW PROCESSLIST;           # MySQL
SELECT * FROM pg_stat_activity WHERE state = 'active';  # PostgreSQL

# Thread dump (Java)
jstack <pid>

# Network latency
ping <db-host>
traceroute <downstream-service>

# File descriptors / connection counts
lsof -p <pid> | wc -l
ss -s
```

### Key Insight
> **Response time = compute time + wait time.** CPU only measures compute. Profile your I/O dependencies — that's where the time usually hides.

---

## 8. Auto Scaling Triggered Correctly, but Performance Still Poor

**What bottleneck may exist?** Auto Scaling adds compute, but if the bottleneck isn't compute, more instances don't help.

### Root Causes to Investigate

- **Database bottleneck:** All new instances hammering one DB — connection exhaustion
- **Shared cache contention:** Redis maxing out connections or memory
- **Message queue saturated:** Kafka/RabbitMQ throughput ceiling reached
- **Scaling lag:** New instances take 3–5 min to become healthy — outage window in between
- **Session stickiness:** Load balancer routes all requests from a session to one (old) instance
- **Cold start problem:** New instances have cold cache — worse performance initially
- **Scaling on wrong metric:** Scaling on CPU but bottleneck is memory or network

### Fix Strategy

```
1. Identify the actual bottleneck with distributed tracing
2. If DB: add read replicas, increase connection pool, use PgBouncer
3. If cache: scale Redis cluster, add read replicas
4. If lag: use predictive scaling or scheduled scaling
5. Scale on the right metric — latency or queue depth, not just CPU
```

### Key Insight
> **Horizontal scaling only helps if your bottleneck is stateless compute.** Map your full dependency graph before tuning auto-scaling policies.

---

## 9. Rollback Completed Successfully, but Issue Still Continues

**What does it mean?** The issue is **not in the application code** — it lies outside what a rollback touches.

### Root Causes to Investigate

- **Database migration not rolled back:** Schema change (dropped column, altered type) persists even after code rollback
- **Stateful data corruption:** Bad data written during the broken deployment remains in DB/cache
- **Infra change:** A separate Terraform change, security group update, or config change wasn't reverted
- **Cache poisoning:** Stale/corrupt data in Redis/Memcached — old app reads it too
- **Third-party dependency:** External service changed its API or is still down
- **Feature flag:** Flag enabled independently of code, still active after rollback
- **CDN cache:** Old broken responses cached at edge — needs purge

### Rollback Checklist

```bash
# 1. Confirm the correct image is running
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# 2. Check if DB migrations need manual rollback
# Run your migration tool's down command

# 3. Flush cache
redis-cli FLUSHDB  # or targeted key deletion

# 4. Purge CDN cache
# AWS CloudFront: create invalidation
# Cloudflare: purge cache via API

# 5. Review all changes in the same release window — not just app code
git log --oneline  # app changes
terraform show     # infra changes
```

### Key Insight
> **A rollback reverts code, not state.** Any side effects (DB writes, migrations, cache, config, infra) require explicit, separate reversal.

---

## 10. DNS Working Internally but Not Externally

**What will you verify?** DNS resolution is context-dependent — internal and external resolvers are entirely separate systems.

### Root Causes to Investigate

- **Split-horizon DNS:** Internal DNS zone returns private IP; external DNS not configured or wrong
- **TTL not propagated:** DNS record changed but old TTL hasn't expired globally
- **Missing public DNS record:** Record exists only in internal (Route 53 private zone, CoreDNS) but not in public zone
- **Firewall blocking port 53:** External DNS queries blocked by security group or ACL
- **NS record misconfiguration:** Domain's name server records point to wrong authoritative server
- **DNSSEC issue:** Signature validation failure blocking external resolution

### Verification Steps

```bash
# Test internal vs external resolution
nslookup myapp.internal           # internal resolver
nslookup myapp.example.com 8.8.8.8  # external (Google DNS)
dig myapp.example.com @1.1.1.1     # external (Cloudflare DNS)

# Check authoritative name servers
dig NS example.com

# Trace full DNS resolution path
dig +trace myapp.example.com

# Check propagation globally
# Use: https://dnschecker.org
```

### Key Insight
> **Internal DNS and public DNS are separate namespaces.** Always test with an external resolver (`8.8.8.8`, `1.1.1.1`) to simulate what real users experience.

---

## 11. SSL Certificate Valid, but HTTPS Still Failing

**Why?** Certificate validity is just one requirement for a working TLS handshake — many others can fail.

### Root Causes to Investigate

| Issue | Description |
|-------|-------------|
| **Chain incomplete** | Intermediate/root CA cert missing from bundle |
| **CN/SAN mismatch** | Cert issued for `www.example.com`, request uses `api.example.com` |
| **Protocol mismatch** | Server only supports TLS 1.3, client requires TLS 1.2 |
| **Cipher suite mismatch** | No common cipher between client and server |
| **Port misconfiguration** | HTTPS listener on 443 not started or bound incorrectly |
| **Cert loaded incorrectly** | Nginx/load balancer config points to wrong cert file |
| **HSTS preload issue** | Browser enforces HTTPS but redirect is misconfigured |
| **Mutual TLS (mTLS)** | Client cert required but not provided |

### Diagnostic Commands

```bash
# Test TLS handshake details
openssl s_client -connect example.com:443 -showcerts

# Check certificate SAN fields
openssl x509 -in cert.pem -text -noout | grep -A1 "Subject Alternative"

# Test with specific TLS version
curl -v --tlsv1.2 https://example.com
curl -v --tlsv1.3 https://example.com

# Verify cert chain
openssl verify -CAfile chain.pem cert.pem
```

### Key Insight
> **A valid certificate doesn't guarantee a successful TLS connection.** Verify the full chain, SNI configuration, protocol versions, and cipher compatibility.

---

## 12. Kubernetes Ingress Configured Properly, but Traffic Blocked

**What else can fail?** Ingress is just a resource definition — the actual traffic flow involves many more components.

### Root Causes to Investigate

- **Ingress Controller not installed:** The Ingress resource exists, but no controller (nginx-ingress, Traefik, etc.) is running to act on it
- **IngressClass mismatch:** `spec.ingressClassName` doesn't match the controller's class
- **Network Policy blocking:** A `NetworkPolicy` blocks traffic between ingress controller pods and app pods
- **Service selector mismatch:** Ingress routes to a Service, but that Service's selector doesn't match pod labels
- **Backend port mismatch:** Service exposes port `80` but pod actually listens on `8080`
- **TLS secret missing or wrong:** HTTPS ingress can't find the referenced TLS secret
- **Namespace issue:** Ingress and Service in different namespaces without cross-namespace config

### Diagnostic Steps

```bash
# Check ingress controller is running
kubectl get pods -n ingress-nginx

# Describe the ingress — look for warning events
kubectl describe ingress <ingress-name>

# Verify the service and endpoints
kubectl get endpoints <service-name>

# Test pod → pod connectivity bypassing ingress
kubectl exec -it <pod> -- curl http://<service-name>:<port>

# Check network policies
kubectl get networkpolicy -A
```

### Key Insight
> **Ingress = intention. Ingress Controller = execution.** Without a running, correctly configured controller, the resource does nothing.

---

## 13. Logs Available, but Root Cause Still Unclear

**What additional data is needed?** Logs alone capture what happened — not *why* or *where in the chain* it broke.

### What to Add

| Data Type | Tool | What It Reveals |
|-----------|------|-----------------|
| **Distributed traces** | Jaeger, Zipkin, AWS X-Ray | Which service in the chain added latency or failed |
| **Metrics correlation** | Prometheus + Grafana | Exact time of anomaly, affected dimensions |
| **Structured logging** | Add request IDs, user IDs, trace IDs | Correlate a single request across services |
| **Profiling** | pprof (Go), async-profiler (Java), py-spy (Python) | CPU/memory hotspots not visible in logs |
| **Database query logs** | Slow query log, `EXPLAIN ANALYZE` | Which queries are expensive |
| **Network captures** | `tcpdump`, Wireshark | Packet-level errors, retransmissions |
| **Error grouping** | Sentry, Rollbar | Aggregate similar errors, show first occurrence |

### Questions That Reveal Missing Instrumentation

```
- Can I trace a single failing user request end-to-end?
- Do I know which downstream call caused the timeout?
- Can I correlate a log entry with a metric spike?
- Do I have the request payload that triggered the error?
- Can I reproduce this with a specific user ID or session?
```

### Key Insight
> **Logs are a record of events. Traces are a record of causality.** If logs don't show root cause, add distributed tracing and structured log correlation.

---

## 14. CI/CD Pipeline Fast, but Deployments Unstable

**What process issue exists?** Speed without safety creates fragility. Fast pipelines that skip verification steps ship bugs faster.

### Root Causes to Investigate

- **Missing or weak test coverage:** Unit tests pass but integration/E2E tests don't exist
- **No staging gate:** Deployments go straight to prod without a staging validation step
- **Concurrent deployments:** Two engineers merge and trigger simultaneous deploys — race condition
- **No canary or blue-green:** 100% traffic switched instantly — failures affect all users
- **Config/secret changes untracked:** Code deploys fine but environment config changed separately
- **Missing rollback automation:** Failed deploys require manual intervention
- **No deployment freeze / change control:** Deploys happen during peak traffic or on Fridays

### Stability Improvements

```yaml
# Example: GitHub Actions with staged gates
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: pytest tests/ --cov=. --cov-fail-under=80

  deploy-staging:
    needs: test
    steps:
      - run: kubectl apply -f k8s/ -n staging
      - run: kubectl rollout status deployment/app -n staging
      - run: ./scripts/smoke-test.sh staging

  deploy-prod:
    needs: deploy-staging
    environment: production   # Requires manual approval
    steps:
      - run: kubectl apply -f k8s/ -n production
```

### Key Insight
> **Pipeline speed is a vanity metric if it skips validation.** Stability requires gates: test → staging → smoke test → canary → production.

---

## 15. Terraform State Correct, but Cloud Resources Drifted

**How?** Drift occurs when resources are modified **outside of Terraform** — manually, via console, CLI, or other automation.

### Root Causes to Investigate

- **Manual console changes:** Someone edited a security group, added an S3 bucket policy, or resized an instance directly
- **Other automation tools:** A Lambda, runbook, or Chef/Ansible script modified the resource
- **Cloud provider auto-remediation:** AWS Config rules, Service Control Policies, or auto-scaling modified resources
- **Resource replacement with same ID:** Resource was deleted and manually recreated — Terraform doesn't know
- **Shared resources:** Resource managed by multiple Terraform workspaces — last-write wins

### Detection and Remediation

```bash
# Detect drift
terraform plan  # Shows what would change = current drift

# Refresh state from real cloud (updates state, not resources)
terraform refresh

# Import manually created resources into state
terraform import aws_instance.web i-1234567890abcdef0

# Prevent drift: use Terraform Cloud or AWS Config for drift detection alerts

# Enforce no manual changes with SCPs or IAM policies
# Require all infra changes through PR → pipeline → terraform apply
```

### Key Insight
> **Terraform state is a snapshot, not a live mirror.** Drift is inevitable without enforcing IaC as the exclusive path for infrastructure changes.

---

## 16. Redis Cache Healthy, but App Still Slow

**Why?** Redis being "up" doesn't mean it's being used effectively — or at all.

### Root Causes to Investigate

- **Cache miss storm:** Cache was flushed/restarted — all requests hit the DB simultaneously (thundering herd)
- **Cache not being hit:** Wrong key format, serialization mismatch, or TTL too short
- **Hot key problem:** Single key accessed millions of times/sec — Redis single-threaded bottleneck
- **Memory eviction:** Redis evicting keys under memory pressure — effective hit rate drops
- **Network latency to Redis:** Redis healthy but slow round-trips from app server
- **Large values:** Storing huge objects in Redis causing serialization/deserialization overhead
- **Pipeline not used:** App makes 100 sequential Redis calls instead of one pipelined batch

### Diagnostic Commands

```bash
# Check hit rate
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# Monitor real-time commands
redis-cli MONITOR

# Check memory usage and eviction
redis-cli INFO memory | grep -E "used_memory_human|evicted_keys"

# Find hot keys
redis-cli --hotkeys

# Check client connections
redis-cli INFO clients
```

### Key Insight
> **Cache health ≠ cache effectiveness.** Monitor your hit rate — if it's below 80%, your cache isn't helping. Investigate what's being missed and why.

---

## 17. Kafka Brokers Healthy, but Consumer Lag Increasing

**What could cause it?** Consumer lag = producers are faster than consumers. The brokers are just the pipe — the problem is at one of the ends.

### Root Causes to Investigate

- **Consumer too slow:** Processing logic is computationally expensive or calls slow downstream services
- **Rebalancing too frequent:** Consumers joining/leaving triggers rebalances — pause processing
- **Single-partition topic:** Only one consumer can read one partition — no parallelism possible
- **Consumer group misconfigured:** Too few consumer instances relative to partition count
- **Downstream dependency slow:** Consumer writes to DB or calls API that becomes the bottleneck
- **Deserialization errors:** Consumer failing silently on bad messages — retrying endlessly
- **Max poll records too high:** Consumer fetches too many records, exceeds `max.poll.interval.ms`, gets kicked

### Diagnostic Steps

```bash
# Check consumer lag per partition
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group my-consumer-group

# Watch lag over time
watch -n 5 kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group my-consumer-group

# Check partition count vs consumer count
kafka-topics.sh --describe --topic my-topic --bootstrap-server broker:9092

# Review consumer config
max.poll.records=500          # lower this if processing is slow
max.poll.interval.ms=300000   # ensure processing fits within this
```

### Key Insight
> **Lag is a consumer problem, not a broker problem.** Scale consumers up to partition count, optimize processing, and check for downstream bottlenecks.

---

## 18. Nginx Healthy, but Backend Services Overloaded

**Why?** Nginx is a highly efficient proxy that can handle thousands of concurrent connections — it's almost never the bottleneck. It can funnel more traffic than your backends can handle.

### Root Causes to Investigate

- **No rate limiting:** Nginx accepts all traffic and passes it all upstream — backends saturate
- **Upstream connection pooling misconfigured:** Too many connections opened to backends
- **No circuit breaker:** Failed backends keep receiving traffic — pile-on failure
- **Uneven load distribution:** Sticky sessions or poor hashing routes all traffic to a few backends
- **Missing timeouts:** Slow backend requests hold open Nginx upstream connections — queue builds
- **Traffic spike not handled:** Bots, scrapers, or a marketing campaign floods traffic

### Nginx Configuration Fixes

```nginx
upstream backend {
    server app1:8080 max_fails=3 fail_timeout=30s;
    server app2:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;  # Connection pooling
}

server {
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req zone=api burst=20 nodelay;

    location /api/ {
        proxy_pass http://backend;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;       # Don't hold connections forever
        proxy_next_upstream error timeout; # Failover on error
    }
}
```

### Key Insight
> **Nginx is a fire hose. Your backend is a garden hose.** Add rate limiting, timeouts, and circuit breaking at the Nginx layer to protect your services.

---

## 19. Alerts Triggered Late During Outage

**What monitoring mistake exists?** Reactive monitoring — alerting on **symptoms** (errors, downtime) instead of **leading indicators** (saturation, latency trends).

### Common Monitoring Mistakes

| Mistake | Better Approach |
|---------|-----------------|
| Alert on errors only | Alert on error *rate increase* — catch it before it's widespread |
| Alert on downtime | Alert on latency > P95 threshold — systems degrade before they die |
| Single threshold alerts | Use anomaly detection — alert when behavior deviates from baseline |
| Alert on averages | Alert on **percentiles** (P95, P99) — averages hide tail latency |
| No burn rate alerts | Use SLO burn rate alerts — detect fast-burn incidents early |
| Missing leading indicators | Monitor queue depth, connection pool usage, disk growth rate |

### The Four Golden Signals (Google SRE)

```
1. Latency     — How long does a request take? (P95/P99, not average)
2. Traffic     — How many requests per second?
3. Errors      — What percentage of requests are failing?
4. Saturation  — How full is your system? (CPU, memory, disk, connections)
```

### Alert Improvement Checklist

```
[ ] Alert on SLO burn rate, not just raw error rate
[ ] Set alerts on P95/P99 latency, not average
[ ] Add leading indicator alerts (queue depth, connection pool %)
[ ] Reduce alert noise — group related alerts, suppress during maintenance
[ ] Define severity levels — not everything is PagerDuty at 3AM
[ ] Test your alerts regularly — fire drills / chaos engineering
```

### Key Insight
> **Late alerts mean you're measuring the wrong things.** Alert on leading indicators. By the time your error rate spikes, users have already noticed.

---

## 20. "Everything Looks Healthy" but Production is Failing

**What is your approach?** This is the hardest scenario — it means your observability has blind spots. Here is a systematic playbook.

### Step-by-Step Approach

#### Phase 1: Confirm & Characterize (0–5 min)
```
1. Reproduce the failure yourself — don't rely solely on customer reports
2. Quantify scope: how many users? which regions? which features?
3. Check recent changes: deployments, config changes, infra changes (last 24h)
4. Identify the first report time — correlate with deployment timeline
```

#### Phase 2: Expand Observability (5–15 min)
```
5. Move beyond dashboards — query raw metrics, not aggregates
6. Check error rates by endpoint, not overall average
7. Review distributed traces for the failing user flows
8. Check third-party status pages (payment, auth, CDN, cloud provider)
9. Look for patterns: geography, user segment, specific request type
10. Check database: slow queries, lock waits, replication lag
```

#### Phase 3: Isolate the Layer (15–30 min)
```
11. Test each layer in isolation:
    Browser/client → Load Balancer → Service A → Service B → Database
12. Use synthetic transactions to pinpoint the breaking step
13. Check network: packet loss, DNS, TLS handshake times
14. Review feature flags — is a flag active for affected users?
15. Check queue depths, cache hit rates, connection pool usage
```

#### Phase 4: Mitigate First, Investigate Second
```
16. If a recent change is suspected — rollback immediately, investigate later
17. Enable circuit breakers to prevent cascade
18. Redirect traffic to healthy region/instance if possible
19. Disable the failing feature via feature flag
20. Communicate status to customers — transparency builds trust
```

### The Systematic Checklist for "Green but Broken"

```
Infrastructure Layer:   CPU ✓ | Memory ✓ | Disk ✓ | Network ✓
Application Layer:      Error rate by endpoint | Latency P95/P99 | Thread pools
Data Layer:             DB query times | Replication lag | Cache hit rate
External Dependencies:  Third-party APIs | CDN | Auth providers | Payment
User Layer:             RUM data | Specific user segments | Geographic region
Recent Changes:         Code deploys | Config changes | Infra changes | Feature flags
```

### Key Insight
> **"Everything is green" means your monitoring is not asking the right questions.** After every incident where this happens, add a new metric, trace, or alert that would have caught it earlier. Each blind spot found is an investment in future reliability.

---

## 🎯 Summary: The Meta-Skill Being Tested

All 20 questions test the same underlying skill:

```
Observed state ≠ Actual state
Tool output ≠ User experience
One layer passing ≠ Full stack working
```

| Principle | Application |
|-----------|-------------|
| **Think in layers** | Every system has multiple layers — check each one |
| **Correlate, don't assume** | Find the timeline, find the change, find the scope |
| **Distinguish signal from noise** | Not all metrics measure what matters to users |
| **Separate detection from causation** | A healthy component can mask an unhealthy dependency |
| **Mitigate, then investigate** | Restore service first; do RCA after |
| **Build observability debt into incidents** | Every surprise = a missing metric or alert |

---

*Prepared for DevOps / SRE / Platform Engineering Interview Preparation*
*Covers: Kubernetes • Jenkins • Docker • Terraform • DNS • SSL • Kafka • Redis • Nginx • CI/CD*
