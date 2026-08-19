# Observability Current-State Survey

## Deliverables

**Scope:** Helm-GitOps repository + inspected Kubernetes cluster state  
**Date:** 19 August 2026  
**Purpose:** Establish the current observability coverage, identify gaps, and recommend a tooling direction.

> **Evidence note:** Findings are limited to the repository/configuration and live cluster outputs inspected so far. Items marked **VERIFY** should not be treated as confirmed until the corresponding command/repository check is completed.

---

# 1. Executive Summary

The inspected environment currently has **log collection through Grafana Promtail and Loki**, and **Kubernetes resource metrics through Metrics Server**.

The main gap is **application-level observability**:

- Application metrics are not currently evident for the DeepIQ application services.
- No Tempo, Jaeger, or OpenTelemetry deployment was identified.
- No distributed tracing path was identified.
- Loki is using a filesystem backend with a 10 Gi PVC, and no explicit retention configuration was identified in the inspected Loki ConfigMap.
- Promtail is deployed as a DaemonSet and is the current log collection agent.
- Alerting requires additional verification because Prometheus/Alertmanager were not confirmed from the live pod inventory supplied.
- Airflow observability requires an additional repository/live check before being stated as a confirmed gap.

**Overall assessment:** The current environment provides basic infrastructure/log visibility, but it does not yet provide complete application observability across logs, metrics, traces, and alerting.

---

# 2. Current Observability Stack

| Signal / Component | Current State | Evidence | Assessment |
|---|---|---|---|
| Logs | Promtail → Loki | Promtail DaemonSet with 3 ready pods; Loki StatefulSet running | Confirmed |
| Log storage | Loki filesystem | `object_store: filesystem`, `/var/loki/chunks` | Confirmed |
| Log retention | No explicit retention setting identified | Loki ConfigMap inspection | Gap / Verify |
| Kubernetes metrics | Metrics Server | `metrics-server` pod running in `kube-system` | Confirmed |
| Application metrics | No application scrape annotations identified; no app ServiceMonitors found in inspected repo | kubectl + GitOps inspection | Gap |
| Traces | No Tempo/Jaeger/OTel deployment identified | kubectl checks | Gap |
| OpenTelemetry | No OTel deployment/configuration identified in cluster | kubectl checks | Gap |
| Alerting | Prometheus/Alertmanager not confirmed in supplied live inventory | kubectl inventory | Verify |
| Log agent | Promtail 3.0.0 | Promtail DaemonSet | Migration candidate |
| Grafana | Not confirmed from supplied live pod inventory | kubectl inventory | Verify |

---

# 3. System Inventory

## 3.1 Application workloads observed

| Namespace | Workload examples | Runtime / framework evidence | Observability evidence |
|---|---|---|---|
| `datastudio` | gateway-service, platform-service, user-management-service, asset-service, databrowser-service, file-browser-service, git-service | Container image names identify services; GitOps repo contains application charts | No application scrape annotations found |
| `datastudio` | reactjs-app, nextjs-app | React / Next.js identified from workload/image names | No application scrape annotations found |
| `datastudio` | airflow-service | Airflow image/configuration present | Metrics path requires verification |
| `bizcontext` | backend-service, frontend-service | Application-specific images | No application scrape annotations found |
| `drilluminatibackend` | django-service, celery-service, celery-beat-service | Django/Python identified from workload names and image naming | No application scrape annotations found |
| `drilluminatidemointernal` | django-service, celery-service, celery-beat-service | Django/Python identified | No application scrape annotations found |
| `streamlit-3535353535` | streamlitappdjango-service | Streamlit/application image | No application scrape annotations found |
| `keycloak` | Keycloak + PostgreSQL | Keycloak / PostgreSQL | Infrastructure/exporter monitoring requires verification |
| `loki` | Loki, caches, gateway, canary | Grafana Loki 3.4.2 | Loki itself exposes metrics; collection path requires Prometheus verification |
| `promtail` | Promtail DaemonSet | Grafana Promtail 3.0.0 | Agent metrics endpoint exists |

## 3.2 Platform / infrastructure components

Observed components include:

- AWS VPC CNI
- CoreDNS
- kube-proxy
- EBS CSI
- EFS CSI
- Metrics Server
- Ingress NGINX
- cert-manager
- External DNS
- External Secrets
- Keycloak
- Loki
- Promtail

---

# 4. Log Observability

## Current path

```text
Application Pods
      |
      | stdout / stderr
      v
/var/log/pods
      |
      v
Promtail DaemonSet
      |
      v
Loki
      |
      v
Filesystem-backed storage
```

### Evidence

Promtail runs as:

- DaemonSet: `datastudio-promtail-dev`
- Desired/current/ready: `3/3/3`
- Image: `grafana/promtail:3.0.0`
- Config mounted from Secret: `datastudio-promtail-dev`
- Host paths mounted:
  - `/var/log/pods`
  - `/var/lib/docker/containers`

Loki runs as:

- StatefulSet: `datastudio-loki-dev`
- Image: `grafana/loki:3.4.2`
- Storage PVC: `10Gi`
- Storage class: `gp2`
- Object store: `filesystem`
- Chunk directory: `/var/loki/chunks`

### Gap

No explicit Loki retention configuration was identified in the inspected ConfigMap.

**Risk:** filesystem-backed Loki storage has a finite PVC capacity. Without an intentional retention policy and capacity strategy, storage growth can eventually become an operational risk.

---

# 5. Metrics Observability

## Current state

Kubernetes Metrics Server is running:

```text
kube-system/metrics-server-7b478c6458-rp96q
```

However, Metrics Server is not a replacement for a Prometheus-based application metrics platform.

The inspected application pods did not expose the standard Prometheus scrape annotations. Only the following pod was returned by the annotation check:

```text
cert-manager
  prometheus.io/scrape=true
  prometheus.io/port=9402
  prometheus.io/path=/metrics
```

The GitOps repository inspection also found that the application charts did not contain application-level ServiceMonitor/PodMonitor resources.

### Gap

The current evidence indicates:

```text
Kubernetes resource metrics     → available
Application RED metrics         → not evident
Application business metrics    → not evident
Prometheus collection           → VERIFY
```

Examples of missing application metrics:

- HTTP request rate
- HTTP error rate
- HTTP latency
- request duration percentiles
- queue depth
- job failures
- application-specific business metrics

---

# 6. Distributed Tracing

The following searches returned no matching workloads/configuration:

```bash
kubectl get pods -A | grep -Ei 'tempo|jaeger|opentelemetry|otel'

kubectl get pods -A -o yaml | grep -Ei 'OTEL_|opentelemetry|jaeger|tempo|traceparent|tracing'
```

### Finding

No distributed tracing backend or OpenTelemetry deployment was identified in the inspected cluster.

Therefore:

```text
Traces = Not currently evident
```

### Impact

Cross-service troubleshooting remains dependent on manually correlating timestamps, pod logs, and request information.

---

# 7. Alerting and Prometheus

No Prometheus, Alertmanager, or Grafana pods were detected in the inspected EKS cluster:

```bash
kubectl get pods -A | grep -Ei 'prometheus|alertmanager|grafana'
```
No output was returned.

The Kubernetes API also does not expose the Prometheus Operator ServiceMonitor resource:

```bash
kubectl get servicemonitor,podmonitor -A
```
returned:
error: the server doesn't have a resource type "servicemonitor"

The API resource check also did not show Argo CD Application or ApplicationSet resources:

```bash
kubectl api-resources | grep -Ei 'application|applicationset'
```
Only an unrelated ApplicationNetworkPolicy resource was returned.

### Finding

Prometheus-based metrics collection, Alertmanager-based alerting, and Grafana were not detected in the inspected EKS cluster.

This should be interpreted as a finding about the inspected cluster, not proof that these components do not exist elsewhere.

Scope limitation: Prometheus/Grafana/Alertmanager may exist in another cluster, namespace, account, or external/managed environment. The current evidence only establishes that they were not detected through the inspected cluster context.

---

# 8. Airflow Observability

The GitOps repository contains an Airflow deployment and references to Python/Airflow metrics functionality.

Airflow observability is currently not active in the inspected EKS cluster. The GitOps configuration explicitly disables both Airflow OTel traces (tracesEnabled: false) and OTel metrics (metricsEnabled: false). No StatsD or OTel Collector pod was detected in the cluster, and the Kubernetes API does not expose the ServiceMonitor resource. No Prometheus instance was detected either. The Airflow service and workload are running, but there is currently no verified metrics or tracing collection path for Airflow.

---

# 9. Gap Analysis

| Priority | Gap | Impact | Recommended action |
|---|---|---|---|
| P1 | Application metrics not evident | Cannot measure application error rate/latency/traffic reliably | Add Prometheus-compatible metrics / OTel metrics |
| P1 | Alerting path not confirmed | Potentially no actionable notification path | Verify Prometheus + Alertmanager and configure receivers |
| P1 | Loki retention/storage strategy incomplete | Risk of storage exhaustion | Configure retention and move durable chunks/index data to S3 |
| P1 | No distributed tracing | Slow cross-service troubleshooting | Introduce OpenTelemetry + Tempo |
| P2 | Promtail is legacy | Long-term support/migration risk | Plan Promtail → Grafana Alloy |
| P2 | Application instrumentation inconsistent | Limited service-level visibility | Standardize OTel instrumentation |
| P2 | Airflow collection path unclear | Important workload may have uncollected metrics | Verify and onboard Airflow |
| P3 | Observability ownership/routing unclear | Alerts may not reach the correct team | Define severity and on-call routing |
| P3 | Production/QA parity not confirmed | Coverage may differ between environments | Inventory each cluster/environment |

---

# 10. Tooling Options

## Option A — Managed observability platform

Examples:

- Grafana Cloud
- Datadog
- New Relic

### Advantages

- Lower operational burden
- Managed storage
- Managed upgrades
- Faster rollout
- Built-in alerting and dashboards

### Disadvantages

- Recurring usage cost
- Vendor dependency
- Data leaves the cluster/infrastructure
- Compliance/security review may be required
- Application instrumentation is still required

---

## Option B — Consolidate the existing self-hosted approach — Recommended

Standardize on:

```text
Metrics     → Prometheus
Logs        → Loki
Traces      → Tempo
Visualization → Grafana
Agent       → Grafana Alloy
Instrumentation → OpenTelemetry
Alerting    → Alertmanager
Storage     → S3 for durable Loki/Tempo data
```

### Advantages

- Reuses existing Loki/Grafana ecosystem where already deployed
- Lower recurring SaaS cost
- Full control over telemetry data
- OpenTelemetry keeps instrumentation vendor-neutral
- Prometheus/LogQL/Grafana skills remain reusable

### Disadvantages

- Team owns upgrades
- Team owns storage/capacity
- Requires operational expertise
- Additional compute/storage required for metrics and traces

---

## Option C — Hybrid / OTel-first

Use OpenTelemetry instrumentation and collectors now while deferring the final backend decision.

### Advantages

- Vendor-neutral instrumentation
- Keeps managed/self-hosted options open

### Disadvantages

- Does not by itself solve retention, alerting, dashboards, or storage
- Can become an indefinite "later" decision

---

# 11. Recommendation

## Recommended: Option B — Consolidate the self-hosted stack

The recommended target is:

```text
OpenTelemetry instrumentation
          |
          +------ Metrics ------> Prometheus
          |
          +------ Logs ---------> Alloy ------> Loki
          |
          +------ Traces -------> Alloy ------> Tempo
                                      |
                                      v
                                   Grafana
                                      |
                                      v
                                 Alertmanager
                                      |
                                Slack / Email
```

### Why

The currently identified problems are primarily **coverage and configuration gaps**, not evidence that the existing observability ecosystem is fundamentally unsuitable.

The recommended sequence is:

1. Verify Prometheus/Grafana/Alertmanager state.
2. Fix alert routing.
3. Configure Loki retention.
4. Move Loki durable storage to S3.
5. Introduce OpenTelemetry instrumentation.
6. Deploy Tempo.
7. Add application metrics.
8. Migrate Promtail to Alloy.
9. Extend coverage to Airflow and other clusters.
10. Standardize dashboards and alert rules.

---

# 12. Rough Cost Estimate

These are **planning estimates, not quotes**. Actual cost depends heavily on telemetry volume, retention, and compute requirements.

## Self-hosted

| Item | Planning estimate |
|---|---:|
| S3 for 1 TB Loki data | ~US$23/month storage at the commonly published S3 Standard rate; requests/transfer are additional |
| 100 GB S3 | ~US$2.30/month storage |
| Loki existing PVC | Existing infrastructure cost; current PVC is only 10Gi |
| Tempo compute | Additional EKS compute; exact cost depends on sizing |
| Prometheus compute/storage | Additional/expanded EKS resources depending on retention and scrape volume |
| Alloy | DaemonSet; generally adds CPU/memory consumption rather than a separate SaaS charge |
| **Expected incremental range** | **Tens to low hundreds of USD/month for a small deployment**, excluding existing EKS node costs and significant telemetry growth |

AWS currently lists gp3 EBS storage around $0.08/GB-month in applicable regions, while gp2 is around $0.10/GB-month. citeturn1search1turn1search7

## Grafana Cloud

Grafana Cloud Pro currently starts at $19/month plus usage. Grafana documents usage-based pricing for logs/traces and other telemetry; for new customers, logs/traces are charged according to processing/writing/retention usage. citeturn0search8turn0search10

Illustrative example only:

- 100 GB/month processed logs
- 100 GB/month written logs
- 30-day retention
- plus metrics/traces

This would be materially more than the $19 platform fee, and exact cost requires actual telemetry volume.

## Datadog

Datadog's current published pricing includes Infrastructure Pro at $15/host/month when billed annually, APM at $31/host/month, and separate log ingestion/indexing charges. citeturn0search5

For example, **10 infrastructure hosts** at the annual Infrastructure Pro rate would be approximately:

```text
10 × $15 = $150/month
```

before APM, logs, custom metrics, and other products.

### Cost conclusion

Do **not** make a final managed-platform cost claim until actual daily log volume and metric/trace volume are measured.

---

# 13. Open Questions

1. What is the actual log ingestion volume per day?
2. How much Loki data is currently stored?
3. Is Prometheus installed in this cluster or another cluster?
4. Is Grafana installed and where?
5. Is Alertmanager installed?
6. Who owns alert/on-call routing?
7. Which channels should receive critical alerts?
8. Is customer telemetry allowed to leave DeepIQ infrastructure?
9. Does production/QA have the same observability configuration?
10. What retention period is required for logs, metrics, and traces?

---

# 14. Proposed Follow-up Work

1. **Verify current Prometheus/Grafana/Alertmanager deployment**
2. **Configure Alertmanager receivers and severity routing**
3. **Configure Loki retention**
4. **Move Loki durable storage to S3**
5. **Standardize ServiceMonitor/PodMonitor labels**
6. **Instrument one application as an OpenTelemetry pilot**
7. **Deploy Tempo**
8. **Add application metrics dashboards**
9. **Verify and onboard Airflow metrics**
10. **Migrate Promtail to Grafana Alloy**
11. **Define observability standards for all application charts**
12. **Repeat inventory for QA/production clusters**

---

# 15. Final Assessment

The current environment has a useful starting point:

- Loki is deployed.
- Promtail is deployed.
- Kubernetes Metrics Server is deployed.
- Application workloads are clearly identifiable.
- The GitOps repository provides a path to standardize observability configuration.

However, the environment is not yet providing complete application observability.

The highest-value improvement is to establish a standard:

**OpenTelemetry + Prometheus + Loki + Tempo + Grafana + Alertmanager**, with **Grafana Alloy** as the collection agent.

The final backend decision should remain reversible. OpenTelemetry instrumentation makes a later move to Grafana Cloud, Datadog, or another backend substantially easier.
