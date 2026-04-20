# Prometheus Fundamentals - Interview Prep Notes

---

## OVERVIEW & ARCHITECTURE FUNDAMENTALS

### What is Prometheus?
Prometheus is an open-source time-series monitoring and alerting toolkit designed for microservices environments. It uses a **pull-based model** where it periodically scrapes metrics from instrumented applications and infrastructure components, storing them in its local time-series database (TSDB).

### Core Design Principles
1. **Reliable**: Single Prometheus server should be fully functional; minimal dependencies
2. **Scalable**: Horizontal scaling via federation; long-term storage via remote backends
3. **Push vs Pull**: Prometheus pulls metrics (better for reliability and multi-tenancy); Pushgateway available for batch jobs
4. **Operational simplicity**: Service discovery, no complex distributed setup required for basic deployment

### Why Pull Model Over Push?
- **Reliability**: Prometheus can detect when scraping fails (explicit vs implicit)
- **Load control**: Prometheus controls scrape frequency and timeout
- **Debugging**: Can hit `/metrics` endpoint directly for troubleshooting
- **Multi-tenancy**: Single exporter can expose metrics for multiple Prometheus instances

---

## 1. PROMETHEUS ARCHITECTURE

### Core Components
```
┌─────────────────────────────────────────────────────────────┐
│                      Applications/Services                   │
│  (Expose /metrics endpoint with Prometheus metrics)          │
└────────────────────────┬────────────────────────────────────┘
                         │ Scrape
┌────────────────────────▼────────────────────────────────────┐
│                  Prometheus Server                           │
│  ┌──────────────────────────────────────┐                    │
│  │  TSDB (Time Series Database)         │                    │
│  │  Stores metrics with timestamps      │                    │
│  └──────────────────────────────────────┘                    │
│  ┌──────────────────────────────────────┐                    │
│  │  Query Engine (PromQL)               │                    │
│  │  Evaluates queries against TSDB      │                    │
│  └──────────────────────────────────────┘                    │
│  ┌──────────────────────────────────────┐                    │
│  │  Alert Engine                        │                    │
│  │  Evaluates alert rules               │                    │
│  └──────────────────────────────────────┘                    │
└────────┬───────────────────────────┬───────────────────────────┘
         │ Sends Alerts              │ Query & Visualize
         ▼                           ▼
    ┌──────────────┐        ┌──────────────────┐
    │ AlertManager │        │ Grafana / UI     │
    │ (Routes,     │        │ (Dashboards)     │
    │  notifies)   │        └──────────────────┘
    └──────────────┘
```

### Key Points
- **Pull Model**: Prometheus actively scrapes targets (not pushed to)
- **Time Series Database**: Stores metrics with timestamps
- **Local Storage**: Default is local disk (can add remote storage)
- **Scraping**: Periodically fetches `/metrics` from each target

---

## 2. SERVICE DISCOVERY

### What It Is
Service Discovery (SD) is the automated mechanism for finding and registering monitoring targets dynamically. Instead of static `scrape_configs`, Prometheus queries a service registry (Kubernetes API, Consul, DNS, etc.) to obtain target lists and update them in real-time.

### Problem Service Discovery Solves
- **Scale**: Manual target lists don't scale with container orchestration (auto-scaling groups, dynamic pod creation)
- **Operational overhead**: Updating config files on every deployment is error-prone
- **Failure detection**: Removes targets when they become unavailable
- **Multi-tenancy**: Enables automatic discovery across multiple teams/namespaces

### Types
| Type | Use Case | Example |
|------|----------|---------|
| **Static Configs** | Small, fixed environments | Dev/test with 5 servers |
| **Kubernetes SD** | K8s clusters | Auto-discover pods, services |
| **Consul SD** | Service mesh | Consul-registered services |
| **DNS SD** | DNS records | Route53, etcd |
| **AWS EC2 SD** | EC2 instances | Auto-scaling groups |
| **File-based SD** | Custom automation | Dynamically updated JSON |

### Kubernetes Service Discovery (Interview Focus)
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

**How it works**:
1. Prometheus queries K8s API
2. Gets list of all pods/services/nodes
3. Dynamically adds/removes targets
4. Uses relabeling to customize metrics

### Interview Answer
> "Service Discovery automatically finds targets to monitor. In Kubernetes, Prometheus queries the API server to discover pods and services. This eliminates manual configuration and adapts when services scale."

---

## 3. PROMQL (Prometheus Query Language)

### What It Is
PromQL is Prometheus's functional query language for filtering, aggregating, and transforming time-series data. It supports instant queries (current value) and range queries (time window) with operators for arithmetic, comparison, aggregation, and rate calculations.

### Fundamental Concepts

**Instant Vector**: Current value of time series (single point in time)
```promql
http_requests_total                    # Returns current value of all matched series
```

**Range Vector**: Time series values over a time window
```promql
http_requests_total[5m]               # Returns all values in last 5 minutes
```

**Scalar**: Single numeric value (result of functions)
```promql
count(up)                              # Returns number of instances
```

**String**: Rarely used, returned by some functions

### Metric Types (Prometheus Exposition Format Types)

**Counter**: Monotonically increasing metric; values never decrease except on reset
- TSDB stores raw counter value
- Use `rate()` or `increase()` for analysis (converts to gauge-like behavior)
- Examples: `http_requests_total`, `errors_total`, `bytes_processed_total`
- Queries: `rate(counter[5m])` for per-second rate, `increase(counter[1h])` for total increase

**Gauge**: Instantaneous measurement that can increase or decrease
- Represents current state; no rate calculation needed
- Examples: `node_memory_MemFree_bytes`, `container_cpu_usage_seconds_total`, queue length
- Queries: Direct aggregation without `rate()` function

**Histogram**: Samples observations and counts them in buckets; tracks sum and count
- Enables distribution analysis (percentiles, quantiles)
- Auto-creates `_bucket`, `_count`, `_sum` suffixes
- Examples: `http_request_duration_seconds`, `request_size_bytes`
- Queries: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`

**Summary**: Pre-computed quantiles; similar to histogram but with worse accuracy
- Quantiles computed on client-side (not queryable from server)
- Uses: When you need specific percentiles without post-aggregation
- Generally prefer Histogram over Summary for flexibility

### Common PromQL Operations

**Rate Calculation**: Convert counter to per-second measurement
```promql
rate(http_requests_total[5m])          # Requests per second over 5-min window
increase(http_requests_total[1h])      # Total increase in 1 hour
```

**Aggregation**: Combine time series by labels
```promql
sum(http_requests_total) by (method)   # Sum by HTTP method
topk(5, http_requests_total)           # Top 5 highest values
avg(cpu_usage_percent)                 # Average across instances
count(up)                              # Count matching series
```

**Filtering**: Select series based on label matching
```promql
http_requests_total{job="api-server"}           # Exact match
http_requests_total{status=~"5.."}              # Regex match (5xx errors)
http_requests_total{environment!="test"}        # Negation
```

**Arithmetic & Comparison**:
```promql
(rate(errors[5m]) / rate(requests[5m])) * 100  # Error percentage
memory_usage > 1024*1024*1024                   # Greater than 1GB
```

### Common Patterns
```promql
# Error rate percentage
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100

# Requests per second per endpoint
sum(rate(http_requests_total[5m])) by (endpoint)

# Percentage of successful requests
count(http_requests_total{status=~"2.."}) / count(http_requests_total) * 100
```

### Interview Answer
> "PromQL is Prometheus's query language. It selects metrics by name and labels, calculates rates of change, and aggregates data. Most queries use `rate()` for counters and `by()` for grouping by labels."

---

## 4. EXPORTERS

### Architectural Role
An exporter is a process that collects metrics from a target system (application, service, OS, database) and exposesmétrics in Prometheus text format on an HTTP `/metrics` endpoint. Prometheus then scrapes this endpoint like any other target.

### The Exporter Problem-Solution Pattern

**Problem**: Legacy systems, databases, and services without Prometheus instrumentation
**Solution**: External process polls system → translates to Prometheus format → HTTP endpoint

### Implementation Patterns

#### 1. Official/Third-Party Exporters
Pre-built exporters for common systems:
- Node Exporter (OS metrics), MySQL Exporter (database), Redis Exporter (cache)
- PostgreSQL Exporter, Nginx Exporter, and 100+ community exporters
- Deployment: Run as separate process/container pointing at target system

#### 2. Custom Exporters
For proprietary/niche systems:
- Process connects to target (API, SQL, shell commands)
- Periodically queries system state
- Converts to Prometheus metric format
- Exposes HTTP `/metrics` endpoint
- Languages: Go (preferred), Python, Rust

#### 3. Built-in Instrumentation (Preferred)
Application directly instrumented:
- Uses Prometheus client library (prometheus-net .NET, prometheus-client Python, etc.)
- Code directly records metrics (counters, gauges, histograms)
- App natively exposes `/metrics` endpoint
- No separate process needed
- Best practice for applications you control

### Trade-Offs

| Pattern | Pros | Cons |
|---------|------|------|
| **Official Exporter** | Maintained, feature-rich | Separate process, network latency, may not expose all metrics |
| **Custom Exporter** | Full control, business logic | Maintenance burden, extra service to monitor, SPOF |
| **Built-in** | Low latency, application insights | Requires code changes, limited library support per language |

---

## 5. CUSTOM METRICS

### Definition & Purpose
Custom metrics are application-specific measurements tracking business logic and operational state. Unlike infrastructure metrics (CPU, memory), custom metrics directly reflect application behavior and enable observability of SLA/SLO targets.

### Instrumentation Implementation (.NET)

```csharp
// Initialize metrics
var todoCounter = new Counter<int>(
    name: "todos_created_total",
    description: "Total todos created"
);

var todoGauge = new UpDownCounter<int>(
    name: "todos_pending",
    description: "Current pending todos"
);

var latencyHistogram = new Histogram<double>(
    name: "todo_request_duration_seconds",
    description: "Request latency in seconds",
    unit: "s"
);

// Record metrics
todoCounter.Add(1, new KeyValuePair<string, object>("priority", "high"));
todoGauge.Set(pending_count);
latencyHistogram.Record(stopwatch.Elapsed.TotalSeconds);
```

### Best Practices

**Naming Convention**: `<namespace>_<subsystem>_<name>_<unit>`
```
✓ todo_api_todos_created_total
✓ server_http_request_duration_seconds
✓ database_query_latency_milliseconds
✗ counter1, metric_x (ambiguous)
```

**Label Selection**:
- Use for bounded dimensions (method, endpoint, status code) → 5-20 unique values
- Avoid high-cardinality labels (user_id, request_id, IP address) → millions of values
- Cardinality explosion: 1M users × 100 endpoints = 100M time series

**Metric Type Selection**:
- **Counter**: Only increases, for events/totals (requests, errors, logins)
- **Gauge**: Current state, can increase/decrease (queue length, active connections, memory)
- **Histogram**: Distribution in buckets, enables percentile queries (latency, size, duration)
- **Summary**: Pre-computed quantiles, deprecated in favor of Histogram

### Cardinality Management

Cardinality = number of unique time series. High cardinality causes memory exhaustion, query slowness, and storage growth.

**Mitigation Strategies**:
1. Avoid unbounded labels (no user IDs, IP addresses)
2. Use `metric_relabel_configs` with `action: drop` to exclude high-cardinality labels before storing
3. Monitor cardinality: query `prometheus_tsdb_metric_chunks_created`
4. Aggregate at scrape time rather than exporter time
5. Use label restrictions in client library configuration

---

## 6. PUSHGATEWAY

### Architectural Purpose
Pushgateway is an intermediary service that accepts metrics pushed via HTTP, buffers them with TTL semantics, and exposes a `/metrics` endpoint for Prometheus to scrape. It addresses scenarios where the pull model is impractical.

### Problem Solved
Pull model requires the exporter to be addressable and running when Prometheus scrapes. This fails for:
- Batch jobs that complete before Prometheus scrapes
- One-off/ephemeral tasks
- Systems outside Prometheus network (firewall/NAT issues)

### Architecture Tradeoffs

**Pull Model (Preferred)**:
- ✓ Prometheus detects target failure (scrape timeout)
- ✓ Prometheus controls scrape rate and load
- ✓ Single authority (Prometheus)
- ✗ Exporter must be long-running

**Push Model with Pushgateway**:
- ✓ Batch jobs can push and exit immediately
- ✗ Pushgateway becomes single point of failure
- ✗ Unclear if job failed or succeeded (old metrics remain)
- ✗ Multiple Prometheus instances push duplicate data
- ✗ Deduplication complexity in HA setups
- ✗ Metrics stale but still exposed

### Pushgateway Internals

**Metric Lifecycle**:
1. Job pushes metrics to `/metrics/job/<job_name>`
2. Pushgateway stores with instance label
3. Metrics expire after TTL (default: no TTL, must manually delete)
4. Prometheus scrapes `/metrics` and fetches all stored metrics

**HTTP Endpoints**:
```bash
# Push metrics
curl -X POST --data-binary @metrics.txt http://pushgateway:9091/metrics/job/backup_job

# Manual deletion
curl -X DELETE http://pushgateway:9091/metrics/job/backup_job

# View stored metrics  
curl http://pushgateway:9091/metrics
```

### When to Use
- Batch/cron jobs (backup, cleanup, reports)
- Ephemeral containers in Kubernetes
- Legacy systems that only support push
- External monitoring sources

### When NOT to Use
- Long-running services (use pull)
- High-volume metric sources (use pull)
- HA requirements (Pushgateway not horizontally scalable)
- Critical metrics (failure detection)

---

## 7. ALERTMANAGER

### Architectural Role
Alertmanager is a separate component that receives alert notifications from Prometheus, applies deduplication/grouping logic, and routes them to notification channels (email, Slack, PagerDuty, etc.). It provides a complete alerting workflow independent of Prometheus itself.

### Alert Flow

```
Prometheus (rules engine)
    ↓ evaluates alert rules every 30s
Alert fires (e.g., high cpu)
    ↓ HTTP POST to Alertmanager
Alertmanager (grouping, routing)
    ↓ deduplicates, groups related alerts
Route decision (severity match?)
    ↓
┌─────────────────────────┐
├─ Slack (team notification)
├─ PagerDuty (on-call)
├─ Email (historical)
├─ Webhook (custom integration)
├─ Opsgenie (alert aggregation)
└─────────────────────────┘
```

### Key Responsibilities

**Deduplication**: If same alert fires multiple times, send notification once
**Grouping**: Combine related alerts (20 disk errors → 1 notification group)
**Routing**: Different rules route to different receivers (critical → PagerDuty, warning → Slack)
**Silencing**: Temporarily suppress alerts during maintenance
**Inhibition**: Suppress triggered alerts if parent alert is already firing

### Configuration Components

**Global Settings**:
```yaml
global:
  resolve_timeout: 5m  # Alert auto-resolves if not updated
  slack_api_url: 'WEBHOOK_URL'
```

**Routing** (alert decision tree):
```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster']  # Group by these labels
  group_wait: 10s    # Wait 10s for more alerts before sending
  group_interval: 5m # Resend after 5m of new alerts
  repeat_interval: 12h # Repeat resolved alerts every 12h
  
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
    - match_re:
        job: 'prod-.*'
      receiver: 'prodteam'
```

**Receivers** (notification targets):
```yaml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - routing_key: 'KEY'
  - name: 'slack-platform'
    slack_configs:
      - channel: '#platform-alerts'
        api_url: 'WEBHOOK'
```

**Silencing** (runtime suppression):
```
Suppress all alerts with job=backup for next 2 hours
→ New backup failures won't trigger notifications
```

**Inhibition** (conditional suppression):
```yaml
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['cluster', 'service']
# Translation: Don't send warning alerts if critical alert exists for same cluster/service
```

### Integration Points

**Prometheus Config** (tells Prometheus where to send):
```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

**Webhook Format** (custom receivers):
```json
{
  "alerts": [
    {
      "labels": {
        "alertname": "HighErrorRate",
        "severity": "critical",
        "service": "api"
      },
      "annotations": {
        "summary": "Error rate > 5%",
        "description": "api service error rate is 8.5%"
      }
    }
  ]
}
```

### Common Patterns

**Severity-Based Routing**:
```
DEBUG, INFO: Slack (dev channel)
WARNING: Slack (on-call channel)
CRITICAL: PagerDuty + SMS (immediate escalation)
```

**Environment-Based Routing**:
```
labels=env:staging → Slack #staging-alerts
labels=env:prod → PagerDuty + Email
```

**Team-Based Routing**:
```
match job='frontend-.*' → frontend team Slack
match job='backend-.*' → backend team Slack
```

---

## 8. TSDB (Time Series Database)

### Architecture & Purpose
Prometheus's local TSDB is a specialized database optimized for storing time-series data (metric name + labels → values over time). It prioritizes write optimization (high ingest rate) and query performance over random access.

### Core Data Structure

**Time Series**: A sequence of (timestamp, value) pairs with unique metric name and label set
```
http_requests_total{method="GET", endpoint="/api/users"} 
  [1713360000, 100]
  [1713360030, 115]
  [1713360060, 128]
  ...
```

**Cardinality**: Total unique time series = combinations of (metric name × unique label values)
```
Metric: http_requests_total
Labels: method (GET, POST = 2), status (200, 400, 500 = 3)
Cardinality: 1 × 2 × 3 = 6 series
```

### Storage Layout

**Directory Structure**:
```
/prometheus/
├── wal/                    # Write-Ahead Log (uncommitted writes)
│   ├── 000000
│   ├── 000001
│   └── checkpoint          # Checkpoint for faster recovery
├── chunks_head/            # Recent data in memory (last 2-3 hours)
│   └── 000001              # Block file
└── <block_id>/             # Completed time blocks (one per 2 hours)
    ├── index               # Labels + series lookup index
    ├── meta.json           # Block metadata (min/max time, stats)
    ├── chunks/
    │   ├── 000001          # Compressed chunk data
    │   └── 000002
    └── tombstones          # Deleted series markers
```

### Block & Chunk Organization

**Block**: Contains 2-3 hours of data (default 2h)
- Immutable once completed
- Indexed for fast queries
- Compressed (typically 1 byte per sample)

**Chunk**: Sub-block compression unit within a block
- Stores 1000+ samples
- Enables fast time range queries without decompressing entire block
- XOR compression for float values

### Metrics & Retention

**Data Retention** (two options):
```yaml
# Option 1: Time-based
storage.tsdb.retention.time=15d  # Keep 15 days of data

# Option 2: Size-based
storage.tsdb.retention.size=50GB # Keep max 50GB
```

**Retention Behavior**:
- Prometheus deletes oldest blocks when exceeding retention
- Blocks are deleted atomically (no partial deletion)
- Query returns empty if asking for data past retention window

### Performance Characteristics

**Ingest Rate**: 1 million samples/second typical per instance
**Query Latency**: Milliseconds for most queries over 1 month history
**Compression**: ~1 byte per sample (with timestamps)
**Memory Usage**: ~50KB per 1M samples in head block

### Limitations & Scaling

**Single Instance Limitations**:
- Data availability: No replication (single disk failure = data loss)
- Storage: 50 GB typical on single server (2 weeks of high-volume metrics)
- Availability: No built-in HA

**Scaling Solutions**:
- **For long-term storage**: Remote storage backends (object storage)
- **For HA + long-term**: Thanos (Prometheus sidecar + object storage)
- **For federation**: Multiple Prometheus instances + Thanos

### Authentication & Backup

**Backup Strategy**:
- Stop Prometheus (pauses writes)
- Copy `/prometheus` directory to backup
- Restart Prometheus
- No live backup method without stopping writes

**Recommended Approach**:
- Use Thanos for production (handles backup, HA, retention)
- Use remote storage (AWS S3, GCS) for backup
- Stream metrics to long-term store asynchronously

## 9. SYSTEM DESIGN & SCALING

### Single Instance Architecture

```
Application/Service (instrumented)
    ↓ /metrics endpoint
Prometheus Server
    ├─ Scraper (fetches metrics)
    ├─ TSDB (stores locally - 15d default)
    └─ Query Engine (PromQL evaluation)
         ↓
Alertmanager (alert routing)
         ↓
Notification channels (Slack, PagerDuty, email)

Grafana (visualization layer, separate)
    ↓ queries
Prometheus
```

### High Availability Deployment

**Problem**: Single Prometheus is SPOF (single point of failure)
**Solution**: Multiple independent Prometheus replicas + Alertmanager HA

```
┌─ Target A ──┐
├─ Target B ──┼─→ Prometheus 1 ─→ Alertmanager 1 ─┐
├─ Target C ──┤                                     │
└──────────────┘                                     → Deduplication → Notifications

┌─ Target A ──┐                                     │
├─ Target B ──┼─→ Prometheus 2 ─→ Alertmanager 2 ─┘
├─ Target C ──┤
└──────────────┘

Alertmanager Cluster (3+ instances)
    ↓ gossip protocol (mesh networking)
    - Deduplicates duplicate alerts from multiple Prometheus
    - Ensures one notification sent even if multiple Prometheus fire
```

### Scaling for Long-Term Storage

**Prometheus Limitation**: 2-3 weeks of data typical on single server

**Solution: Thanos Architecture**
```
Prometheus 1 (2h blocks) ─→ Thanos Sidecar ──→ Object Storage (S3, GCS)
Prometheus 2 (2h blocks) ─→ Thanos Sidecar ─→ (unlimited history)
Prometheus 3 (2h blocks) ─→ Thanos Sidecar ─→

                            ↓
                    Thanos Query Layer
                    (queries across all Prometheus + object storage)
                            ↓
                        Grafana
```

**Thanos Benefits**:
- Global view across multiple Prometheus instances
- Object storage for unlimited retention (cheap)
- Query deduplication (same metric from multiple sources)
- Downsampling (1s resolution → 1d resolution for old data)
- No local disk scaling needed

### Scrape Configuration Optimization

**Scrape Job Design**:
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    scrape_interval: 30s     # How often to scrape
    scrape_timeout: 10s      # Timeout per scrape
    kubernetes_sd_configs:
      - role: pod
    metric_relabel_configs:  # Filter metrics before storing
      - source_labels: [__name__]
        regex: 'temp_.*'
        action: drop         # Don't store temp metrics
```

**Common Mistakes**:
- Too frequent scraping (30s minimum recommended, 15s okay)
- Too many metrics per target (filter with `metric_relabel_configs`)
- High cardinality labels (unbounded user IDs, IP addresses)

---

## 10. INTERVIEW Q&A

### Q: "Explain Prometheus architecture and how it differs from other monitoring systems"

**A**: Prometheus is a pull-based time-series monitoring system designed for microservices and Kubernetes. Key architectural differences:

1. **Pull vs Push**: Prometheus pulls metrics from targets via scraping (instead of agents pushing). Advantages: simpler deployment, Prometheus controls load, failure detection via scrape timeout
2. **Time-Series DB**: Local TSDB optimized for metrics (not events/logs), with compression and label-based indexing
3. **Service Discovery**: Built-in SD for Kubernetes, Consul, DNS, AWS EC2 - eliminates static config
4. **PromQL**: Functional query language with native aggregation, rate calculation, and time-window operations
5. **Single-Binary**: No dependencies (except optional remote storage), easy deployment compared to ELK or Datadog

**Not designed for**: Logs (use Loki), traces (use Jaeger), or events (use event streaming)

### Q: "How would you design a monitoring solution for a microservices application in Kubernetes?"

**A**: Multi-component design:

**Metrics Collection**:
- Prometheus with Kubernetes service discovery (Pod/Node/Service roles)
- Scrape config with relabel rules to filter/enrich labels
- Node Exporter for infrastructure metrics (per node)
- Application instrumentation (prometheus-net/client) for custom metrics

**Storage & Retention**:
- Prometheus default (15d) for recent data
- Thanos sidecar for long-term storage to S3/GCS
- Downsampling for older data (1y resolution)

**Alerting**:
- Alert rules in Prometheus (evaluates every 30s)
- Multiple Alertmanager instances in HA (gossip network)
- Route to Slack (warnings), PagerDuty (critical)
- Silence rules for maintenance windows

**Visualization**:
- Grafana with Prometheus data source
- Pre-built dashboards (Kubernetes, application-specific)
- Panel alerts for real-time thresholds

**Scaling Considerations**:
- Multiple Prometheus replicas (HA) scraping same targets
- Thanos Query for global view across replicas
- Cardinality management (relabel to drop high-cardinality labels)
- Recording rules to pre-aggregate expensive queries

### Q: "What's cardinality and how is it problematic?"

**A**: Cardinality is the number of unique time series a metric generates. Each unique combination of (metric name + label values) = one series.

**Example**:
```
http_requests_total{method="GET",status="200",endpoint="/api/users"}  → 1 series
http_requests_total{method="GET",status="200",endpoint="/api/todos"}  → 1 series
http_requests_total{method="POST",status="200",endpoint="/api/users"} → 1 series
...
```

**The Problem** (High Cardinality):
- 1M users × 1K endpoints × 10 status codes = 10B series (IMPOSSIBLE)
- Each series stored separately = memory explosion
- Query performance degrades (scan more series)
- Alerting becomes slow (evaluate 10B series)
- Storage costs skyrocket

**Example of Cardinality Explosion**:
```
Bad:   http_requests_total{user_id="123", request_id="abc123"} → millions of series
Good:  http_requests_total{service="api", method="POST"} → ~20 series
```

**Mitigation**:
1. Never use unbounded labels (user_id, request_id, IP address)
2. Use `metric_relabel_configs` to drop high-cardinality labels before storing
3. Monitor cardinality: `prometheus_tsdb_metric_chunks_created` metric
4. Use static labels per service (not per request)

### Q: "How would you monitor a third-party API you don't control?"

**A**: Use **Blackbox Exporter** (external probe-based monitoring):

```yaml
# Prometheus config
scrape_configs:
  - job_name: 'blackbox-api'
    static_configs:
      - targets:
          - https://api.example.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter:9115  # Exporter address

# Blackbox probes HTTP and records:
# - probe_success (1 = up, 0 = down)
# - probe_duration_seconds (latency)
# - probe_http_status_code (response code)
# - probe_http_ssl_earliest_cert_expiry (SSL cert expiry)
```

**Advantages**:
- No agent needed on remote API
- Measures from your network (validates connectivity)
- Records latency, SSL expiry, response codes
- Creates synthetic transactions

**Metrics Generated**:
```promql
probe_success{instance="https://api.example.com/health"} = 1
probe_duration_seconds{} = 0.234
probe_http_status_code{} = 200
```

**Alerting**:
```
alert: APIDown
expr: probe_success{job="blackbox-api"} == 0
for: 5m  # Alert if down for 5 minutes
```

### Q: "Describe your approach to alert design and preventing alert fatigue"

**A**: Alert design for production readiness:

**Alert Rule Principles**:
1. **Actionable**: Should trigger a specific action (not just "observe")
2. **Well-defined thresholds**: Base on SLO/SLA, not arbitrary numbers
3. **Adequate lead time**: Alert before customer impact (CPU > 80%, not > 99%)

**Example Alert**:
```yaml
- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
  for: 5m  # Must breach for 5m before firing (prevents noise)
  labels:
    severity: critical
  annotations:
    summary: "Service {{ $labels.service }} error rate {{ $value | humanizePercentage }}"
    runbook: "https://wiki/runbooks/high-error-rate"
```

**Preventing Alert Fatigue**:
1. **For loops**: Require metric to breach threshold for N minutes before triggering
2. **Grouping**: Combine related alerts (20 disk errors → 1 notification)
3. **Severity levels**: Critical (immediate action) vs Warning (trend monitoring)
4. **Inhibition rules**: Suppress secondary alerts if primary alert fires
5. **Silence periods**: Suppress alerts during known maintenance
6. **Alert thresholds**: Use percentiles (p95 latency) not absolutes (all requests < 1s)

**Example of Prevention**:
```yaml
# DON'T: Alert on every spike
expr: latency_seconds > 1  # Too noisy

# DO: Alert on persistent trend
expr: quantile(0.95, latency_seconds[5m]) > 1
for: 10m  # Must be high for 10 minutes
```

### Q: "What are recording rules and when would you use them?"

**A**: Recording rules pre-compute expensive queries and store results as new metrics. Used for optimization.

```yaml
groups:
  - name: application
    rules:
      # Rule: Calculate error rate every 30s
      - record: instance:request_errors:rate5m
        expr: rate(http_requests_total{status=~"5.."}[5m])
      
      # Rule: Pre-calculate percentile
      - record: instance:request_latency:p95
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**Benefits**:
- Pre-computed queries run faster (avoid calculating on query time)
- Reduce Prometheus query load
- Enable queries on very old data (e.g., p99 latency over 1-year window)

**When to Use**:
- Complex queries used in multiple dashboards
- Expensive calculations (histograms, percentiles)
- Long-term trend analysis (monthly reports)

**When NOT to Use**:
- Simple queries (avoid premature optimization)
- One-off queries (cost not justified)

### Q: "How would you troubleshoot a 'No data returned' error in Prometheus?"

**A**: Systematic debugging:

1. **Check metric exists**:
   ```
   - UI: Graph tab, enter metric name, check "Test" button
   - Or: curl http://prometheus:9090/api/v1/label/__name__/values | grep metric_name
   ```

2. **Check labels are correct**:
   ```
   http_requests_total{service="api"}  # Will this match what exporter exposes?
   ```

3. **Verify target is being scraped**:
   ```
   - Prometheus UI → Targets tab
   - Status should be "UP"
   - If DOWN: Check network, exporter health, firewall
   ```

4. **Check /metrics endpoint directly**:
   ```bash
   curl http://exporter:8080/metrics | grep metric_name
   # Should show lines like: metric_name{labels} value
   ```

5. **Check metric retention**:
   ```promql
   # Is metric older than retention policy?
   time() - max(timestamp(metric_name)) > 15d  # If true, data is deleted
   ```

6. **Check relabel_configs**:
   ```yaml
   # metric_relabel_configs might be dropping the metric
   - source_labels: [__name__]
     regex: 'metric_name'
     action: drop  # Oops! This drops it
   ```

### Q: "How does Prometheus handle NaN and Inf values?"

**A**: 
- **NaN** (Not a Number): Returned when division by zero (`rate(0 / 0)`) or query on non-existent series
- **Inf**: Returned when `x / 0` where x > 0
- **Query behavior**: PromQL drops NaN values during aggregation (`sum()` skips NaN)
- **Alerting**: Comparisons with NaN (`NaN > 5`) evaluate to false

**In Practice**:
```
rate(errors[5m]) / rate(requests[5m])  # If requests=0, result is NaN (good, avoids false alert)
```

### Q: "What's the difference between Federation and Thanos for scaling?"

**A**: 

| Aspect | Federation | Thanos |
|--------|-----------|--------|
| **Purpose** | One Prometheus scrapes metrics FROM another Prometheus | Long-term storage + HA + global queries |
| **Setup** | Simple (Prometheus config) | Complex (sidecar + object storage) |
| **Use Case** | Hierarchical setups (regional → global) | Multi-cluster, long-term storage |
| **Query Dedup** | Manual (duplicates across regions) | Automatic (Thanos deduplicates) |
| **Retention** | Each instance separate | Unified via object storage |
| **Complexity** | Low | High |

**Federation Example** (hierarchy):
```
Regional Prometheus 1 ─┐
Regional Prometheus 2 ─┼→ Global Prometheus (scrapes from regional ones)
Regional Prometheus 3 ─┘
```

**Thanos Example** (HA + Scale):
```
Prometheus 1 + Thanos Sidecar ─┐
Prometheus 2 + Thanos Sidecar ─┼→ Object Storage (S3)
Prometheus 3 + Thanos Sidecar ─┘
                               ↓
                        Thanos Query (unified view)
```

---

## 11. QUICK REFERENCE CHEAT SHEET

### PromQL Functions
```promql
rate(metric[5m])           # Per-second rate
increase(metric[1h])       # Total increase
sum/avg/max/min(metric)    # Aggregation
by() / without()           # Group/exclude labels
histogram_quantile(0.95)   # 95th percentile latency
```

### Alert Rules
```yaml
groups:
  - name: example
    interval: 30s  # How often to evaluate
    rules:
      - alert: HighErrorRate
        expr: rate(errors[5m]) > 0.05
        for: 5m  # Must breach for 5 minutes
        labels:
          severity: critical
```

### Relabeling Actions
```yaml
relabel_configs:
  - action: keep           # Keep matching
  - action: drop           # Drop matching
  - action: replace        # Replace label
  - action: labeldrop      # Delete label
```

---

## 12. HANDS-ON PRACTICE SCENARIOS

### Scenario 1: Monitor API Response Time
```promql
# Get 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### Scenario 2: Find Slowest Endpoints
```promql
topk(5, rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m]))
```

### Scenario 3: Alert on Node Disk Full
```
expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
for: 10m
```

### Scenario 4: Create Dashboard in Grafana
- Add Prometheus data source
- Build panels with PromQL queries
- Set refresh rate and time range
- Add threshold alerts to panels

---

## SUMMARY TABLE

| Component | Purpose | Key Concept |
|-----------|---------|-------------|
| **Service Discovery** | Auto-find targets | Queries K8s API, eliminates hardcoding |
| **PromQL** | Query language | Select, aggregate, calculate rates |
| **Exporters** | Expose metrics | Node Exporter, custom apps, instrumentation |
| **Custom Metrics** | Track application data | Counter, Gauge, Histogram, Summary |
| **Pushgateway** | Receive pushed metrics | For batch jobs & short-lived services |
| **Alertmanager** | Route notifications | Deduplicate, group, send to Slack/PagerDuty |
| **TSDB** | Time-series storage | Local disk, compressed, retention policies |

---

**Last Updated**: April 2026 | **Focus**: Kubernetes & Production Scenarios
