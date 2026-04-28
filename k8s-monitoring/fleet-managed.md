# Kubernetes Monitoring with Grafana Fleet Management
## A Guide to Deploying Collectors via Helm and Managing Telemetry Pipelines via Fleet Management

**Document Version:** 1.0  
**Last Updated:** April 2026  
**Applies To:** Grafana Kubernetes Monitoring Helm Chart v3.x, Grafana Alloy v1.7.2+, Grafana Cloud Fleet Management (GA)

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [The Golden Rule: No Precedence, No Conflict](#the-golden-rule-no-precedence-no-conflict)
4. [⚠️ Critical Warnings](#️-critical-warnings)
5. [Deployment Components Reference](#deployment-components-reference)
6. [Deployment Strategy Options](#deployment-strategy-options)
7. [Helm Chart Configuration (values.yaml)](#helm-chart-configuration-valuesyaml)
8. [Fleet Management Pipeline Reference](#fleet-management-pipeline-reference)
   - [Pipeline 1: Pod Logs (`gke_pod_logs`)](#pipeline-1-pod-logs-gke_pod_logs)
   - [Pipeline 2: Node Logs (`gke_node_logs`)](#pipeline-2-node-logs-gke_node_logs)
   - [Pipeline 3: Cluster Events (`gke_cluster_events`)](#pipeline-3-cluster-events-gke_cluster_events)
   - [Pipeline 4: Application Observability (`gke_application_observability`)](#pipeline-4-application-observability-gke_application_observability)
   - [Pipeline 5: eBPF Profiling (`gke_ebpf_profiling`)](#pipeline-5-ebpf-profiling-gke_ebpf_profiling)
9. [Fleet Management Setup Steps](#fleet-management-setup-steps)
10. [Attribute-Based Pipeline Assignment](#attribute-based-pipeline-assignment)
11. [Self-Monitoring Pipelines](#self-monitoring-pipelines)
12. [v3 Chart Limitations and v4 Migration](#v3-chart-limitations-and-v4-migration)
13. [Troubleshooting](#troubleshooting)
14. [Reference Links](#reference-links)

---

## Overview

This document describes how to deploy Kubernetes monitoring using the **Grafana Kubernetes Monitoring Helm chart** (v3) while managing all telemetry collection pipelines centrally via **Grafana Fleet Management**.

The key principle is a **separation of concerns**:

| Layer | Managed By | Purpose |
|---|---|---|
| Collector Infrastructure | **Helm Chart** | Deploys Alloy DaemonSets, StatefulSets, and Deployments to the cluster |
| Backing Services | **Helm Chart** | Deploys kube-state-metrics, Node Exporter, Kepler, OpenCost |
| Telemetry Pipelines | **Fleet Management** | Defines what data is collected and where it is sent |

This separation gives you the ability to update, toggle, or replace telemetry pipelines at runtime — across your entire fleet — **without redeploying the Helm chart**.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster (GKE)                              │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │               k8s-monitoring Helm Chart (v3)                     │ │
│  │                                                                  │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐            │ │
│  │  │alloy-metrics│  │alloy-singleton│  │  alloy-logs │            │ │
│  │  │ StatefulSet │  │  Deployment  │  │  DaemonSet  │            │ │
│  │  │(1+ replicas)│  │ (1 replica)  │  │(1 per node) │            │ │
│  │  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘            │ │
│  │         │                │                  │                   │ │
│  │  ┌──────┴──────┐  ┌──────┴───────┐  ┌──────┴──────┐            │ │
│  │  │alloy-receiver│  │  (events)   │  │ (pod & node │            │ │
│  │  │  DaemonSet  │  │             │  │    logs)    │            │ │
│  │  └──────┬──────┘  └─────────────┘  └─────────────┘            │ │
│  │         │                                                       │ │
│  │  ┌──────┴──────┐                                               │ │
│  │  │alloy-profiles│                                              │ │
│  │  │  DaemonSet  │                                               │ │
│  │  └─────────────┘                                               │ │
│  │                                                                  │ │
│  │  All collectors register with Fleet Management via remotecfg {}  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
                         │ remotecfg poll (default: 5m)
                         ▼
┌────────────────────────────────────────────────────────────────────────┐
│              Grafana Fleet Management                                  │
│         fleet-management-prod-004.grafana.net                          │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  Remote Configuration Pipelines (assigned by matching attrs)    │  │
│  │  ┌──────────────────┐  ┌───────────────────┐                   │  │
│  │  │  gke_pod_logs    │  │  gke_node_logs    │  → alloy-logs     │  │
│  │  └──────────────────┘  └───────────────────┘                   │  │
│  │  ┌──────────────────┐  ┌───────────────────┐                   │  │
│  │  │gke_cluster_events│  │  gke_app_obs      │  → alloy-singleton│  │
│  │  └──────────────────┘  └───────────────────┘     alloy-receiver│  │
│  │  ┌──────────────────┐  ┌───────────────────┐                   │  │
│  │  │ gke_ebpf_profiling│  │ self_monitoring_* │  → alloy-profiles │  │
│  │  └──────────────────┘  └───────────────────┘     (all)        │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Grafana Cloud Destinations                           │
│  Prometheus/Mimir  │  Loki  │  OTLP Gateway  │  Pyroscope            │
└────────────────────────────────────────────────────────────────────────┘
```

---

## The Golden Rule: No Precedence, No Conflict

> **Remote configurations run in parallel with a collector's local configuration. Neither overrides the other.**

Local (Helm) and remote (Fleet Management) configurations each have their own independent component controller. This means:

- ✅ A Fleet Management pipeline error **does not** break local (Helm) configuration
- ✅ Local Helm configuration errors **do not** affect already-running remote pipelines (but may prevent Alloy from starting entirely)
- ✅ Pipelines from different sources run completely isolated from one another
- ✅ You can toggle Fleet Management pipelines on/off without touching the Helm chart
- ❌ Components cannot be **shared** between local and remote configs (e.g., you cannot reference a `loki.write` from the local config in a Fleet Management pipeline)

---

## ⚠️ Critical Warnings

### 1. Duplicate Data = Duplicate Cost

> **This is the most common and costly mistake when combining Helm features with Fleet Management pipelines.**

If a telemetry feature is **enabled in the Helm chart values** (e.g., `podLogs: enabled: true`) **AND** a matching Fleet Management pipeline is also assigned to the same collector, the pipeline runs **twice**. The same data is written **twice** to Grafana Cloud, resulting in:

- 2x log ingestion charges
- Duplicate log records appearing in Loki queries
- Potential label cardinality issues

**Rule:** For each telemetry signal, choose **one owner** — Helm OR Fleet Management. Never both.

```
# ✅ CORRECT — Helm owns pod logs
podLogs:
  enabled: true  # Helm manages this pipeline

# Fleet Management: NO pod_logs pipeline assigned to alloy-logs collectors

---

# ✅ CORRECT — Fleet Management owns pod logs  
podLogs:
  enabled: false  # Helm does NOT generate the pipeline

# Fleet Management: gke_pod_logs pipeline assigned to alloy-logs collectors

---

# ❌ WRONG — Both enabled = duplicate data
podLogs:
  enabled: true   # Helm pipeline running

# Fleet Management: gke_pod_logs pipeline ALSO assigned = data shipped TWICE
```

### 2. The `alloy-logs` DaemonSet Preset is Required for Log File Access

Even when `podLogs: enabled: false` and `nodeLogs: enabled: false` in Helm, the `alloy-logs` collector **must still be deployed with the `filesystem-log-reader` preset**. This preset configures the Alloy Operator to mount these HostPath volumes:

| Mount Path | Contents |
|---|---|
| `/var/log/pods` | Container stdout/stderr logs in CRI format |
| `/var/log/journal` | systemd journal logs (node-level) |

Without these mounts, Fleet Management log pipelines cannot access log files from the node filesystem.

### 3. v3 Backing Services Are Tied to Features

In v3, backing services (kube-state-metrics, Node Exporter, Kepler, OpenCost) are deployed as part of the `clusterMetrics` feature. Setting `clusterMetrics: enabled: false` removes the backing services, leaving Fleet Management metric pipelines with nothing to scrape.

**Workaround for v3:** Keep `clusterMetrics: enabled: true` to deploy backing services, and manage the scraping pipelines via Fleet Management instead.

**Permanent solution:** Migrate to v4, where `telemetryServices` is completely independent of features.

### 4. `logging {}` Blocks Cannot Be Used in Fleet Management Pipelines

Remote configuration does not support top-level configuration blocks. The `logging {}` block cannot be used in Fleet Management configuration pipelines. If you want to change Alloy log levels, you must do so by adding the `logging {}` block to the collector's local configuration (via `extraConfig` in the Helm chart).

### 5. The `alloy-singleton` Must Always Be 1 Replica

The cluster events collector (`alloy-singleton`) watches the Kubernetes API for lifecycle events. Multiple replicas will produce **duplicate event log entries**. This collector must always run as a single-replica Deployment.

### 6. Inactive Collectors After Pod Restarts

For DaemonSet collectors (alloy-logs, alloy-receiver, alloy-profiles), the `GCLOUD_FM_COLLECTOR_ID` includes the node name, which means it is stable across pod restarts on the same node. For singleton/StatefulSet collectors, the ID includes the pod name, which changes on restart, creating a new collector entry in Fleet Management. Old entries are marked inactive after 3 hours and do not count toward your stack's collector limit. Inactive collectors are automatically deleted after 30 days.

---

## Deployment Components Reference

| Component | Workload Type | Key Function | Needs HostPath | Notes |
|---|---|---|---|---|
| `alloy-metrics` | StatefulSet | Scrapes cluster/host/cost/energy metrics | No | Can be scaled with HPA |
| `alloy-singleton` | Deployment | Watches K8s API for lifecycle events | No | **Must stay at 1 replica** |
| `alloy-logs` | DaemonSet | Reads pod & node log files | **Yes** | `filesystem-log-reader` preset required even with features disabled |
| `alloy-receiver` | DaemonSet | Accepts OTLP & Zipkin from instrumented apps | No | Ports must be opened manually when `applicationObservability: false` |
| `alloy-profiles` | DaemonSet | Collects eBPF/Java/pprof profiles | No (but needs root+hostPID) | `privileged` preset required for eBPF |
| `kube-state-metrics` | Deployment | K8s object state metrics | No | Deployed by `clusterMetrics` feature |
| `node-exporter` | DaemonSet | Linux node hardware/OS metrics | Yes | Deployed by `clusterMetrics` feature |
| `kepler` | DaemonSet | Energy metrics via eBPF | No | Deployed by `clusterMetrics` feature |
| `opencost` | Deployment | Infrastructure cost allocation | No | Deployed by `clusterMetrics` feature |
| `alloy-operator` | Deployment | Manages Alloy CRD lifecycle | No | Required for v3+ |

---

## Deployment Strategy Options

### Strategy A: Hybrid (Recommended for v3)

Helm manages all infrastructure **and** cluster/host metrics. Fleet Management manages log, event, trace, and profiling pipelines.

| Feature | Managed By |
|---|---|
| clusterMetrics (+ backing services) | Helm ✅ |
| annotationAutodiscovery | Helm ✅ |
| prometheusOperatorObjects | Helm ✅ |
| podLogs | Fleet Management 🔁 |
| nodeLogs | Fleet Management 🔁 |
| clusterEvents | Fleet Management 🔁 |
| applicationObservability | Fleet Management 🔁 |
| profiling | Fleet Management 🔁 |

### Strategy B: Full Fleet Management

All pipeline features disabled in Helm. All pipelines managed by Fleet Management. Backing services must be deployed separately (or use v4).

### Strategy C: Full Helm

All features enabled in Helm. Fleet Management used only for self-monitoring and health visibility. No custom pipelines created in Fleet Management.

---

## Helm Chart Configuration (values.yaml)

The following `values.yaml` implements **Strategy A** (Hybrid) — Helm deploys all infrastructure, Fleet Management manages log, event, trace, and profile pipelines.

```yaml
# =============================================================================
# values.yaml — GKE Cluster: k8s-monitoring v3 + Fleet Management
#
# Strategy:
#   Helm     → Deploys all collectors and backing services
#              Manages: clusterMetrics, annotationAutodiscovery,
#                       prometheusOperatorObjects
#   Fleet Mgmt → Manages: podLogs, nodeLogs, clusterEvents,
#                          applicationObservability, profiling
#
# IMPORTANT: Features disabled in Helm below must have a corresponding
# Fleet Management pipeline assigned — otherwise no data will flow
# for those signals.
# =============================================================================

cluster:
  name: gke   # Must be unique per cluster — used in collector_id generation

# =============================================================================
# DESTINATIONS
# Keep all destinations in Helm. The chart creates Kubernetes Secrets
# from these credentials. Fleet Management pipelines reference credentials
# via: sys.env("GCLOUD_RW_API_KEY")
# =============================================================================
destinations:
  - name: grafana-cloud-metrics
    type: prometheus
    url: https://prometheus-prod-09-prod-au-southeast-0.grafana.net/api/prom/push
    auth:
      type: basic
      username: "498476"
      password: <METRICS_ACCESS_TOKEN>

  - name: grafana-cloud-logs
    type: loki
    url: https://logs-prod-004.grafana.net/loki/api/v1/push
    auth:
      type: basic
      username: "248208"
      password: <LOGS_ACCESS_TOKEN>

  - name: gc-otlp-endpoint
    type: otlp
    url: https://otlp-gateway-prod-au-southeast-0.grafana.net/otlp
    protocol: http
    auth:
      type: basic
      username: "398905"
      password: <OTLP_ACCESS_TOKEN>
    metrics:
      enabled: true
    logs:
      enabled: true
    traces:
      enabled: true

  - name: grafana-cloud-profiles
    type: pyroscope
    url: https://profiles-prod-009.grafana.net:443
    auth:
      type: basic
      username: "398905"
      password: <PROFILES_ACCESS_TOKEN>

# =============================================================================
# FEATURES — HELM MANAGED
# These features are enabled in Helm because they deploy backing services
# (kube-state-metrics, Node Exporter, Kepler, OpenCost) that Fleet
# Management pipelines depend on. Disabling these would remove backing
# services from the cluster.
# =============================================================================

# KEEP ENABLED — Deploys and scrapes kube-state-metrics, Node Exporter,
# control plane, Kubelet, cAdvisor, Kepler, and OpenCost.
clusterMetrics:
  enabled: true
  opencost:
    enabled: true
    metricsSource: grafana-cloud-metrics
    opencost:
      exporter:
        defaultClusterId: gke
      prometheus:
        existingSecretName: grafana-cloud-metrics-grafana-k8s-monitoring
        external:
          url: https://prometheus-prod-09-prod-au-southeast-0.grafana.net/api/prom
  kepler:
    enabled: true

# KEEP ENABLED — Dynamic annotation-based scraping requires the Helm
# feature to generate the Alloy discovery.relabel rules. Fleet Management
# cannot dynamically watch for pod annotation changes.
annotationAutodiscovery:
  enabled: true

# KEEP ENABLED — Prometheus Operator CRD watching (ServiceMonitor,
# PodMonitor) requires the Helm feature for correct RBAC + configuration.
prometheusOperatorObjects:
  enabled: true

# =============================================================================
# FEATURES — FLEET MANAGEMENT MANAGED
# These features are DISABLED in Helm. The corresponding telemetry will be
# collected via Fleet Management pipelines assigned to the relevant
# collectors.
#
# REMINDER: If a Fleet Management pipeline is not created and assigned for
# each disabled feature, that telemetry will NOT be collected.
# =============================================================================

# DISABLED — Fleet Management pipeline: gke_cluster_events
# Assign to: alloy-singleton (matching attrs: cluster=gke, role=singleton-collector)
clusterEvents:
  enabled: false

# DISABLED — Fleet Management pipeline: gke_pod_logs
# Assign to: alloy-logs (matching attrs: cluster=gke, role=logs-collector)
# NOTE: alloy-logs must still be deployed with filesystem-log-reader preset
# for HostPath mounts to be available.
podLogs:
  enabled: false

# DISABLED — Fleet Management pipeline: gke_node_logs
# Assign to: alloy-logs (matching attrs: cluster=gke, role=logs-collector)
nodeLogs:
  enabled: false

# DISABLED — Fleet Management pipeline: gke_application_observability
# Assign to: alloy-receiver (matching attrs: cluster=gke, role=receiver-collector)
# NOTE: Ports are manually opened in alloy-receiver extraPorts below.
# When this feature is disabled, the Alloy Operator does NOT auto-configure ports.
applicationObservability:
  enabled: false

# DISABLED — Fleet Management pipeline: gke_ebpf_profiling
# Assign to: alloy-profiles (matching attrs: cluster=gke, role=profiles-collector)
# NOTE: The 'privileged' preset still grants root + hostPID access required
# by the eBPF profiler even with this feature disabled.
profiling:
  enabled: false

# =============================================================================
# COLLECTORS — All deployed by Helm
# Pipelines are assigned via Fleet Management (remoteConfig: enabled: true)
# Remove integrations.alloy — Fleet Management auto-generates self_monitoring_*
# =============================================================================

# --- alloy-metrics ---
# StatefulSet: scrapes cluster/host/cost/energy metrics
# Fleet Management matching attrs: cluster=gke, role=metrics-collector
alloy-metrics:
  enabled: true
  alloy:
    extraEnv:
      - name: GCLOUD_RW_API_KEY
        valueFrom:
          secretKeyRef:
            name: alloy-metrics-remote-cfg-grafana-k8s-monitoring
            key: password
      - name: CLUSTER_NAME
        value: gke
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: GCLOUD_FM_COLLECTOR_ID
        value: grafana-k8s-monitoring-$(CLUSTER_NAME)-$(NAMESPACE)-$(POD_NAME)
  remoteConfig:
    enabled: true
    url: https://fleet-management-prod-004.grafana.net
    auth:
      type: basic
      username: "398905"
      password: <FLEET_MANAGEMENT_TOKEN>

# --- alloy-singleton ---
# Single-replica Deployment: cluster events
# Fleet Management matching attrs: cluster=gke, role=singleton-collector
# WARNING: Do NOT scale this above 1 replica — duplicate events will result
alloy-singleton:
  enabled: true
  alloy:
    extraEnv:
      - name: GCLOUD_RW_API_KEY
        valueFrom:
          secretKeyRef:
            name: alloy-singleton-remote-cfg-grafana-k8s-monitoring
            key: password
      - name: CLUSTER_NAME
        value: gke
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: GCLOUD_FM_COLLECTOR_ID
        value: grafana-k8s-monitoring-$(CLUSTER_NAME)-$(NAMESPACE)-$(POD_NAME)
  remoteConfig:
    enabled: true
    url: https://fleet-management-prod-004.grafana.net
    auth:
      type: basic
      username: "398905"
      password: <FLEET_MANAGEMENT_TOKEN>

# --- alloy-logs ---
# DaemonSet (1 pod per Node): pod logs + node journal logs
# Fleet Management matching attrs: cluster=gke, role=logs-collector
#
# CRITICAL: The filesystem-log-reader preset is what tells the Alloy Operator
# to mount /var/log/pods and /var/log/journal via HostPath volumes. These
# mounts are required by the gke_pod_logs and gke_node_logs Fleet Management
# pipelines. Even though podLogs/nodeLogs are disabled as Helm features
# above, these mounts MUST be present for Fleet Management pipelines to work.
alloy-logs:
  enabled: true
  alloy:
    extraEnv:
      - name: GCLOUD_RW_API_KEY
        valueFrom:
          secretKeyRef:
            name: alloy-logs-remote-cfg-grafana-k8s-monitoring
            key: password
      - name: CLUSTER_NAME
        value: gke
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
      - name: GCLOUD_FM_COLLECTOR_ID
        # DaemonSet: collector_id is unique per node (not per pod restart)
        value: grafana-k8s-monitoring-$(CLUSTER_NAME)-$(NAMESPACE)-alloy-logs-$(NODE_NAME)
  remoteConfig:
    enabled: true
    url: https://fleet-management-prod-004.grafana.net
    auth:
      type: basic
      username: "398905"
      password: <FLEET_MANAGEMENT_TOKEN>

# --- alloy-receiver ---
# DaemonSet: receives OTLP (gRPC/HTTP) and Zipkin from instrumented apps
# Fleet Management matching attrs: cluster=gke, role=receiver-collector
#
# IMPORTANT: Since applicationObservability: false above, the Alloy Operator
# will NOT auto-configure ports. extraPorts must be set manually here.
# Without these ports, the Fleet Management pipeline cannot receive traffic.
alloy-receiver:
  enabled: true
  alloy:
    extraPorts:
      - name: otlp-grpc
        port: 4317
        targetPort: 4317
        protocol: TCP
      - name: otlp-http
        port: 4318
        targetPort: 4318
        protocol: TCP
      - name: zipkin
        port: 9411
        targetPort: 9411
        protocol: TCP
    extraEnv:
      - name: GCLOUD_RW_API_KEY
        valueFrom:
          secretKeyRef:
            name: alloy-receiver-remote-cfg-grafana-k8s-monitoring
            key: password
      - name: CLUSTER_NAME
        value: gke
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
      - name: GCLOUD_FM_COLLECTOR_ID
        value: grafana-k8s-monitoring-$(CLUSTER_NAME)-$(NAMESPACE)-alloy-receiver-$(NODE_NAME)
  remoteConfig:
    enabled: true
    url: https://fleet-management-prod-004.grafana.net
    auth:
      type: basic
      username: "398905"
      password: <FLEET_MANAGEMENT_TOKEN>

# --- alloy-profiles ---
# DaemonSet: eBPF, Java, pprof profiling
# Fleet Management matching attrs: cluster=gke, role=profiles-collector
#
# NOTE: The 'privileged' preset (set by the Helm chart for this collector)
# provides root access and host PID namespace. This is required by the
# pyroscope.ebpf component and remains active even with profiling disabled.
alloy-profiles:
  enabled: true
  alloy:
    extraEnv:
      - name: GCLOUD_RW_API_KEY
        valueFrom:
          secretKeyRef:
            name: alloy-profiles-remote-cfg-grafana-k8s-monitoring
            key: password
      - name: CLUSTER_NAME
        value: gke
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
      - name: GCLOUD_FM_COLLECTOR_ID
        value: grafana-k8s-monitoring-$(CLUSTER_NAME)-$(NAMESPACE)-alloy-profiles-$(NODE_NAME)
  remoteConfig:
    enabled: true
    url: https://fleet-management-prod-004.grafana.net
    auth:
      type: basic
      username: "398905"
      password: <FLEET_MANAGEMENT_TOKEN>
```

### Deploying the Chart

```bash
helm upgrade --install \
  grafana-k8s-monitoring \
  grafana/k8s-monitoring \
  --version "^3" \
  --namespace default \
  --create-namespace \
  --values values.yaml
```

> **Note:** Replace all `<*_TOKEN>` placeholders with your actual Grafana Cloud access policy tokens before deploying. These tokens require the `set:alloy-data-write` scope for Fleet Management and write access to your Grafana Cloud telemetry backends.

---

## Fleet Management Pipeline Reference

Create each pipeline via:
**Grafana Cloud → Connections → Collector → Fleet Management → Remote configuration → Create configuration pipeline → Custom configuration**

After saving each pipeline, assign the matching attributes to target the correct collector type.

> **Tip:** Fleet Management has a built-in Alloy configuration syntax validator. After pasting your pipeline content, click **Test configuration pipeline** to validate before saving.

---

### Pipeline 1: Pod Logs (`gke_pod_logs`)

**Target Collector:** `alloy-logs`  
**Matching Attributes:** `cluster=gke` AND `role=logs-collector`

> Reads container stdout/stderr from `/var/log/pods` on each node via the HostPath mount provided by the `filesystem-log-reader` preset.

```alloy
// ============================================================
// Pipeline: gke_pod_logs
// Collector: alloy-logs (DaemonSet — 1 pod per Node)
//
// Prerequisites:
//   - alloy-logs deployed with filesystem-log-reader preset
//   - /var/log/pods HostPath mounted by Alloy Operator
//   - HOSTNAME env var = spec.nodeName (set by DaemonSet preset)
//   - CLUSTER_NAME env var set in values.yaml
//   - GCLOUD_RW_API_KEY env var set from secret
// ============================================================

// Discover all pods running on this specific node only.
discovery.kubernetes "pods" {
  role = "pod"

  selectors {
    role  = "pod"
    field = "spec.nodeName=" + sys.env("HOSTNAME")
  }
}

// Build log file paths and attach Kubernetes metadata labels.
discovery.relabel "pod_logs" {
  targets = discovery.kubernetes.pods.targets

  // Drop non-running pods to avoid tailing stale file handles
  rule {
    source_labels = ["__meta_kubernetes_pod_phase"]
    action        = "drop"
    regex         = "Pending|Succeeded|Failed|Unknown"
  }

  // Build the log file path from pod UID and container name
  rule {
    source_labels = [
      "__meta_kubernetes_pod_uid",
      "__meta_kubernetes_pod_container_name",
    ]
    separator    = "/"
    target_label = "__path__"
    replacement  = "/var/log/pods/*$1/$2/*.log"
  }

  // Standard Kubernetes labels
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
    target_label  = "app"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "node"
  }
  rule {
    replacement  = sys.env("CLUSTER_NAME")
    target_label = "cluster"
  }
}

// Match log files on the node filesystem
local.file_match "pod_logs" {
  path_targets = discovery.relabel.pod_logs.output
}

// Read and tail the matched log files
loki.source.file "pod_logs" {
  targets    = local.file_match.pod_logs.targets
  forward_to = [loki.process.pod_logs.receiver]
}

// Parse CRI log format and attach static labels
loki.process "pod_logs" {
  // Parse containerd/crio log prefix (timestamp, stream, flags)
  stage.cri {}

  stage.static_labels {
    values = {
      cluster = sys.env("CLUSTER_NAME"),
      job     = "integrations/kubernetes/pod-logs",
    }
  }

  forward_to = [loki.write.grafana_cloud_logs.receiver]
}

// Write to Grafana Cloud Loki
loki.write "grafana_cloud_logs" {
  endpoint {
    url = "https://logs-prod-004.grafana.net/loki/api/v1/push"

    basic_auth {
      username = "248208"
      password = sys.env("GCLOUD_RW_API_KEY")
    }
  }
}
```

---

### Pipeline 2: Node Logs (`gke_node_logs`)

**Target Collector:** `alloy-logs`  
**Matching Attributes:** `cluster=gke` AND `role=logs-collector`

> Reads systemd journal entries from `/var/log/journal` on each node via the HostPath mount.

```alloy
// ============================================================
// Pipeline: gke_node_logs
// Collector: alloy-logs (DaemonSet — 1 pod per Node)
//
// Prerequisites:
//   - alloy-logs deployed with filesystem-log-reader preset
//   - /var/log/journal HostPath mounted by Alloy Operator
//   - HOSTNAME env var = spec.nodeName (set by DaemonSet preset)
//   - CLUSTER_NAME env var set in values.yaml
//   - GCLOUD_RW_API_KEY env var set from secret
//
// Note: When running in a container, you must explicitly set
// path = "/var/log/journal" in loki.source.journal. Without
// this, go-systemd uses SD_JOURNAL_LOCAL_ONLY and cannot find
// journal files mounted from the host.
// ============================================================

// Relabeling rules to promote useful journal fields to Loki labels
loki.relabel "node_journal" {
  forward_to = []

  // Promote systemd unit name (e.g. kubelet.service, containerd.service)
  rule {
    source_labels = ["__journal__systemd_unit"]
    target_label  = "unit"
  }

  // Promote process name (comm field)
  rule {
    source_labels = ["__journal__comm"]
    target_label  = "command"
  }

  // Promote transport (journal, stdout, kernel)
  rule {
    source_labels = ["__journal__transport"]
    target_label  = "transport"
  }
}

// Read from the systemd journal — MUST specify path when containerised
loki.source.journal "node_logs" {
  path          = "/var/log/journal"
  relabel_rules = loki.relabel.node_journal.rules
  forward_to    = [loki.process.node_logs.receiver]

  labels = {
    job     = "integrations/node/journal",
    cluster = sys.env("CLUSTER_NAME"),
    node    = sys.env("HOSTNAME"),
  }
}

// Enrich and forward
loki.process "node_logs" {
  stage.static_labels {
    values = {
      cluster = sys.env("CLUSTER_NAME"),
    }
  }

  forward_to = [loki.write.grafana_cloud_logs.receiver]
}

// Write to Grafana Cloud Loki
loki.write "grafana_cloud_logs" {
  endpoint {
    url = "https://logs-prod-004.grafana.net/loki/api/v1/push"

    basic_auth {
      username = "248208"
      password = sys.env("GCLOUD_RW_API_KEY")
    }
  }
}
```

---

### Pipeline 3: Cluster Events (`gke_cluster_events`)

**Target Collector:** `alloy-singleton`  
**Matching Attributes:** `cluster=gke` AND `role=singleton-collector`

> ⚠️ This pipeline **must only be assigned to a single-replica collector**. If it runs on multiple instances, every Kubernetes event is written to Loki multiple times.

```alloy
// ============================================================
// Pipeline: gke_cluster_events
// Collector: alloy-singleton (Single-replica Deployment ONLY)
//
// WARNING: Assigning this pipeline to a DaemonSet or scaled
// Deployment will produce duplicate event logs. It must only
// run on alloy-singleton which is constrained to 1 replica.
//
// Prerequisites:
//   - alloy-singleton deployed and registered with Fleet Mgmt
//   - CLUSTER_NAME env var set in values.yaml
//   - GCLOUD_RW_API_KEY env var set from secret
//   - Alloy service account has ClusterRole to watch events
// ============================================================

// Watch all Kubernetes lifecycle events cluster-wide
// and convert them to structured logfmt log lines.
loki.source.kubernetes_events "cluster_events" {
  job_name   = "integrations/kubernetes/eventhandler"
  log_format = "logfmt"

  forward_to = [loki.process.cluster_events.receiver]
}

// Enrich event log entries with cluster metadata
loki.process "cluster_events" {
  stage.static_labels {
    values = {
      cluster = sys.env("CLUSTER_NAME"),
      job     = "integrations/kubernetes/eventhandler",
    }
  }

  forward_to = [loki.write.grafana_cloud_logs.receiver]
}

// Write to Grafana Cloud Loki
loki.write "grafana_cloud_logs" {
  endpoint {
    url = "https://logs-prod-004.grafana.net/loki/api/v1/push"

    basic_auth {
      username = "248208"
      password = sys.env("GCLOUD_RW_API_KEY")
    }
  }
}
```

---

### Pipeline 4: Application Observability (`gke_application_observability`)

**Target Collector:** `alloy-receiver`  
**Matching Attributes:** `cluster=gke` AND `role=receiver-collector`

> Prerequisites: Ports 4317, 4318, and 9411 must be open on the `alloy-receiver` DaemonSet. These are configured via `extraPorts` in the `values.yaml` when `applicationObservability: enabled: false`.

```alloy
// ============================================================
// Pipeline: gke_application_observability
// Collector: alloy-receiver (DaemonSet)
//
// Receives OTLP (gRPC port 4317, HTTP port 4318) and
// Zipkin (port 9411) from instrumented applications.
//
// Prerequisites:
//   - Ports 4317, 4318, 9411 opened via extraPorts in values.yaml
//   - Applications configured to send to alloy-receiver ClusterIP/NodePort
//   - GCLOUD_RW_API_KEY env var set from secret
// ============================================================

// Accept OTLP over gRPC (port 4317)
otelcol.receiver.otlp "app_obs_grpc" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }

  output {
    metrics = [otelcol.processor.batch.app_obs.input]
    logs    = [otelcol.processor.batch.app_obs.input]
    traces  = [otelcol.processor.batch.app_obs.input]
  }
}

// Accept OTLP over HTTP (port 4318)
otelcol.receiver.otlp "app_obs_http" {
  http {
    endpoint = "0.0.0.0:4318"
  }

  output {
    metrics = [otelcol.processor.batch.app_obs.input]
    logs    = [otelcol.processor.batch.app_obs.input]
    traces  = [otelcol.processor.batch.app_obs.input]
  }
}

// Accept Zipkin traces (port 9411) and convert to OTLP
otelcol.receiver.zipkin "app_obs_zipkin" {
  endpoint = "0.0.0.0:9411"

  output {
    traces = [otelcol.processor.batch.app_obs.input]
  }
}

// Batch telemetry for efficient delivery
otelcol.processor.batch "app_obs" {
  output {
    metrics = [otelcol.exporter.otlphttp.grafana_cloud.input]
    logs    = [otelcol.exporter.otlphttp.grafana_cloud.input]
    traces  = [otelcol.exporter.otlphttp.grafana_cloud.input]
  }
}

// Export all signals to Grafana Cloud OTLP endpoint
otelcol.exporter.otlphttp "grafana_cloud" {
  client {
    endpoint = "https://otlp-gateway-prod-au-southeast-0.grafana.net/otlp"

    auth = otelcol.auth.basic.grafana_cloud.handler
  }
}

otelcol.auth.basic "grafana_cloud" {
  client_auth {
    username = "398905"
    password = sys.env("GCLOUD_RW_API_KEY")
  }
}
```

---

### Pipeline 5: eBPF Profiling (`gke_ebpf_profiling`)

**Target Collector:** `alloy-profiles`  
**Matching Attributes:** `cluster=gke` AND `role=profiles-collector`

> ⚠️ The `pyroscope.ebpf` component requires Alloy to run as **root inside the host PID namespace**. This is handled by the `privileged` preset on `alloy-profiles`.

```alloy
// ============================================================
// Pipeline: gke_ebpf_profiling
// Collector: alloy-profiles (DaemonSet, privileged)
//
// Collects continuous CPU profiles from all containers on this
// node using eBPF — no application instrumentation required.
//
// Prerequisites:
//   - alloy-profiles deployed with privileged preset
//     (provides securityContext.privileged=true + hostPID=true)
//   - Linux kernel >= 4.9 on all profiled nodes
//   - HOSTNAME env var = spec.nodeName (set by DaemonSet preset)
//   - CLUSTER_NAME env var set in values.yaml
//   - GCLOUD_RW_API_KEY env var set from secret
// ============================================================

// Discover all pods running on this specific node
discovery.kubernetes "local_pods" {
  role = "pod"

  selectors {
    role  = "pod"
    field = "spec.nodeName=" + sys.env("HOSTNAME")
  }
}

// Build profiling targets with service_name and Kubernetes metadata
discovery.relabel "profiling_targets" {
  targets = discovery.kubernetes.local_pods.targets

  // Drop non-running pods
  rule {
    action        = "drop"
    regex         = "Succeeded|Failed"
    source_labels = ["__meta_kubernetes_pod_phase"]
  }

  // Build service_name from namespace/container
  rule {
    action        = "replace"
    regex         = "(.*)@(.*)"
    replacement   = "ebpf/${1}/${2}"
    separator     = "@"
    source_labels = [
      "__meta_kubernetes_namespace",
      "__meta_kubernetes_pod_container_name",
    ]
    target_label = "service_name"
  }

  // Attach all pod labels as profiling labels
  rule {
    action = "labelmap"
    regex  = "__meta_kubernetes_pod_label_(.+)"
  }

  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "node"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }
  rule {
    replacement  = sys.env("CLUSTER_NAME")
    target_label = "cluster"
  }
}

// Collect eBPF CPU profiles from matched pods on this node
pyroscope.ebpf "gke_ebpf" {
  targets    = discovery.relabel.profiling_targets.output
  forward_to = [pyroscope.write.grafana_cloud_profiles.receiver]
}

// Write to Grafana Cloud Pyroscope
pyroscope.write "grafana_cloud_profiles" {
  endpoint {
    url = "https://profiles-prod-009.grafana.net:443"

    basic_auth {
      username = "398905"
      password = sys.env("GCLOUD_RW_API_KEY")
    }
  }

  external_labels = {
    cluster = sys.env("CLUSTER_NAME"),
  }
}
```

---

## Fleet Management Setup Steps

### Step 1: Create a Fleet Management API Token

1. Navigate to **Connections → Collector → Configure** in your Grafana Cloud stack
2. Click **Create token** and give it a descriptive name (e.g., `gke-alloy-fleet-token`)
3. The token is pre-configured with the `set:alloy-data-write` scope, which includes:
   - Reading remote configurations from Fleet Management
   - Writing telemetry to Grafana Cloud backends
4. **Copy and save the token securely** — it cannot be viewed again

### Step 2: Deploy the Helm Chart

Replace the `<*_TOKEN>` placeholders in `values.yaml` with your actual tokens, then deploy:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install \
  grafana-k8s-monitoring \
  grafana/k8s-monitoring \
  --version "^3" \
  --namespace default \
  --create-namespace \
  --values values.yaml
```

### Step 3: Verify Collectors Appear in Fleet Management

1. Navigate to **Connections → Collector → Fleet Management → Inventory**
2. You should see entries for each collector pod — one per node for DaemonSets
3. Check the **Status** column — collectors should show as `Active`
4. Auto-generated `self_monitoring_*` pipelines appear in the **Remote configuration** tab

### Step 4: Add Remote Attributes to Collectors

Attribute-based matching requires each collector to have the correct attributes. Add them via the Fleet Management UI:

1. Go to **Inventory** tab
2. Select all `alloy-logs` collectors (filter by `GCLOUD_FM_COLLECTOR_ID` containing `alloy-logs`)
3. Click **Bulk edit attributes**
4. Add:
   - `cluster = gke`
   - `role = logs-collector`
5. Repeat for other collector types with their respective `role` values

> **Alternatively**, add local attributes in the `alloy-logs` `extraConfig` section of `values.yaml`:
> ```yaml
> alloy-logs:
>   alloy:
>     extraConfig: |-
>       remotecfg {
>         attributes = {
>           "cluster" = "gke",
>           "role"    = "logs-collector",
>         }
>       }
> ```
> **Note:** Local attributes in `remotecfg {}` blocks may conflict with the generated `remotecfg {}` block from the Helm chart. Using the Fleet Management UI to set remote attributes is safer for v3.

### Step 5: Create and Assign Fleet Management Pipelines

For each pipeline in the [Fleet Management Pipeline Reference](#fleet-management-pipeline-reference):

1. Go to **Remote configuration → Create configuration pipeline**
2. Select **Custom configuration → Next**
3. Enter a unique pipeline name (e.g., `gke_pod_logs`)
4. Paste the Alloy configuration content
5. Click **Test configuration pipeline** to validate syntax
6. Click **Next** to assign matching attributes
7. Set the matching attributes (e.g., `cluster=gke` AND `role=logs-collector`)
8. Click **Save** to activate the pipeline

### Step 6: Verify Data Flow

After pipeline activation:

1. Allow 1–2 polling cycles for collectors to receive the new configuration (default poll interval: 5 minutes)
2. Check **Fleet Management → Inventory → [collector] → Configuration tab** to confirm the pipeline appears in the collector's effective configuration
3. Navigate to **Grafana Cloud → Explore** and query your Loki/Prometheus/Pyroscope data sources
4. Check **Fleet Management → Inventory → [collector] → Logs tab** for any errors

---

## Attribute-Based Pipeline Assignment

Fleet Management uses collector attributes to determine which pipelines a collector receives. Attributes can be **local** (set in the Alloy `remotecfg {}` block) or **remote** (set via the Fleet Management UI or API).

> Remote attributes take precedence over local attributes when there is a conflict.

### Recommended Attribute Schema

| Collector | `cluster` | `role` |
|---|---|---|
| `alloy-metrics` | `gke` | `metrics-collector` |
| `alloy-singleton` | `gke` | `singleton-collector` |
| `alloy-logs` | `gke` | `logs-collector` |
| `alloy-receiver` | `gke` | `receiver-collector` |
| `alloy-profiles` | `gke` | `profiles-collector` |

### Pipeline-to-Collector Mapping

| Pipeline Name | Matching Attributes | Assigned To |
|---|---|---|
| `gke_pod_logs` | `cluster=gke`, `role=logs-collector` | `alloy-logs` |
| `gke_node_logs` | `cluster=gke`, `role=logs-collector` | `alloy-logs` |
| `gke_cluster_events` | `cluster=gke`, `role=singleton-collector` | `alloy-singleton` |
| `gke_application_observability` | `cluster=gke`, `role=receiver-collector` | `alloy-receiver` |
| `gke_ebpf_profiling` | `cluster=gke`, `role=profiles-collector` | `alloy-profiles` |
| `self_monitoring_metrics` | Auto-assigned | All collectors |
| `self_monitoring_logs_kubernetes` | Auto-assigned | All collectors |

Matching attributes use **AND logic**: all listed attributes must be present on the collector for the pipeline to be assigned. For example, a pipeline with `["cluster=gke", "role=logs-collector"]` is only assigned to collectors that have **both** attributes.

---

## Self-Monitoring Pipelines

When you first visit the Fleet Management interface after deploying your collectors, a set of self-monitoring configuration pipelines is automatically created and assigned.

| Pipeline Name | Function |
|---|---|
| `self_monitoring_metrics` | Collects internal Alloy health metrics (component health, memory, CPU) |
| `self_monitoring_logs_kubernetes` | Collects internal Alloy log output from Kubernetes pod logs |

These pipelines power:
- The **health dashboards** in the collector details view
- The **Logs tab** in the collector details view
- Fleet Management health status indicators

> **Do not delete** these pipelines unless you intend to disable health monitoring. If accidentally deleted, you can regenerate them by deleting all remaining `self_monitoring_*` pipelines — Fleet Management will recreate them all automatically.

The self-monitoring pipelines rely on the following environment variables being set wherever the collector is running:
- `GCLOUD_RW_API_KEY` — API token for writing telemetry
- `GCLOUD_FM_COLLECTOR_ID` — Unique ID matching the `remotecfg` `id` argument
- `NAMESPACE`, `POD_NAME`, `HOSTNAME` — For log correlation

These are all set in the `extraEnv` sections of the `values.yaml` above.

---

## v3 Chart Limitations and v4 Migration

### Current Limitations in v3

| Limitation | Impact | Workaround |
|---|---|---|
| Backing services tied to feature flags | Cannot disable clusterMetrics without removing kube-state-metrics, Node Exporter, etc. | Keep `clusterMetrics: enabled: true` |
| Fixed collector names | Cannot customise collector topology | Accepted limitation in v3 |
| No native `telemetryServices` independence | Cannot deploy backing service separately from feature | Upgrade to v4 |
| Pod log features (Loki vs OTel) mixed in one flag | `gatherMethod` flag changes entire behaviour | Upgrade to v4 for split features |

### v4 Migration

Versions 1 and 2 of the chart are approaching end-of-life (tentatively around June 2026). Version 3 continues to receive dependency updates, but all new features land exclusively in v4.

Key v4 improvements relevant to this use case:

- **Collectors map**: Name your collectors anything. Presets describe the deployment shape explicitly
- **`telemetryServices` independence**: Deploy kube-state-metrics, Node Exporter, Kepler, and OpenCost independently of feature flags
- **Split features**: `clusterMetrics`, `hostMetrics`, and `costMetrics` are now three separate, focused features
- **Explicit feature-to-collector assignment**: No hidden routing — you can see exactly which collector handles which feature

**Migration tool available** — paste your v3 `values.yaml` and receive a v4-compatible file:
```
https://grafana.github.io/k8s-monitoring-helm-migrator/
```

v4 equivalent of the collector block used in this document:
```yaml
# v4 equivalent
collectors:
  metrics-collector:
    presets: [clustered, statefulset]
    remoteConfig:
      enabled: true
      url: https://fleet-management-prod-004.grafana.net
      # ...

  logs-collector:
    presets: [filesystem-log-reader, daemonset]
    remoteConfig:
      enabled: true
      # ...

  events-collector:
    presets: [singleton]
    remoteConfig:
      enabled: true
      # ...

clusterMetrics:
  enabled: true
  collector: metrics-collector

podLogsViaLoki:
  enabled: false  # Managed by Fleet Management

nodeLogs:
  enabled: false  # Managed by Fleet Management
```

---

## Troubleshooting

### Collectors Not Appearing in Fleet Management Inventory

1. Check the collector pod logs for `remotecfg` errors:
   ```bash
   kubectl logs -n default -l app.kubernetes.io/name=alloy-logs --tail=100
   ```
2. Look for `failed to fetch remote configuration from the API` or authentication errors
3. Verify `GCLOUD_RW_API_KEY` matches the access policy token created for Fleet Management
4. Confirm the Fleet Management URL is correct (check **Connections → Collector → Fleet Management → API** tab for your URL)

### No Data After Pipeline Activation

1. Confirm the pipeline is **Active** (green toggle) in Fleet Management → Remote configuration
2. Check that the collector has the correct **matching attributes** (Inventory → collector → Attributes tab)
3. Wait for the next poll cycle (default: 5 minutes). Check the Logs tab for `remote configuration updated successfully`
4. For log pipelines: verify that `/var/log/pods` or `/var/log/journal` is mounted on the `alloy-logs` pod:
   ```bash
   kubectl exec -n default <alloy-logs-pod> -- ls /var/log/pods
   kubectl exec -n default <alloy-logs-pod> -- ls /var/log/journal
   ```

### Duplicate Data in Grafana Cloud

1. Check whether the same signal is enabled **both** in Helm features AND in a Fleet Management pipeline
2. In Helm `values.yaml`: set the relevant feature to `enabled: false`
3. Run `helm upgrade` to apply the change
4. Verify in the collector's **Configuration tab** in Fleet Management that only the remote pipeline is running

### Pipeline Syntax Errors

Fleet Management has a built-in Alloy configuration syntax validator. If Fleet Management detects a problem in the syntax, an error message displays immediately. After clearing all errors, click **Test configuration pipeline** to verify valid syntax.

For errors that only appear at runtime:
1. Go to **Fleet Management → Inventory → [collector] → Logs tab**
2. Look for component-level errors
3. Or check the collector details view — when you click on the collector, you see the Configuration tab and a list of configuration errors, complete with row and column numbers

### `logging {}` Block in Pipeline

Remote configuration does not support top-level configuration blocks. The `logging {}` block cannot be used in Fleet Management configuration pipelines. Add it to the collector's local configuration via `extraConfig` in the Helm chart instead.

### Regenerating Self-Monitoring Pipelines

If self-monitoring pipelines are missing or broken:
1. Delete **all** `self_monitoring_*` pipelines from the Remote configuration tab
2. Wait — Fleet Management recreates them all automatically
3. Allow a few minutes for the collector to poll and receive the new configuration

---

## Reference Links

| Resource | URL |
|---|---|
| k8s-monitoring Helm chart GitHub | https://github.com/grafana/k8s-monitoring-helm |
| Helm chart documentation | https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/ |
| Fleet Management documentation | https://grafana.com/docs/grafana-cloud/send-data/fleet-management/ |
| Onboard Kubernetes collectors to Fleet Management | https://grafana.com/docs/grafana-cloud/send-data/fleet-management/set-up/onboard-collectors/kubernetes/ |
| Fleet Management architecture | https://grafana.com/docs/grafana-cloud/send-data/fleet-management/introduction/architecture/ |
| Configuration deployment strategies | https://grafana.com/docs/grafana-cloud/send-data/fleet-management/set-up/configuration-pipelines/config-deployment-strategies/ |
| GitOps integration with Fleet Management | https://grafana.com/docs/grafana-cloud/send-data/fleet-management/set-up/infrastructure-as-code/gitops/ |
| Alloy loki.source.file | https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.file/ |
| Alloy loki.source.journal | https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.journal/ |
| Alloy loki.source.kubernetes_events | https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes_events/ |
| Alloy otelcol.receiver.otlp | https://grafana.com/docs/alloy/latest/reference/components/otelcol/otelcol.receiver.otlp/ |
| Alloy pyroscope.ebpf | https://grafana.com/docs/alloy/latest/reference/components/pyroscope/pyroscope.ebpf/ |
| v3 → v4 migration guide | https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/configuration/helm-chart-config/helm-chart/migrate-helm-chart/ |
| v3 → v4 migration tool | https://grafana.github.io/k8s-monitoring-helm-migrator/ |
| Grafana Cloud access policies | https://grafana.com/docs/grafana-cloud/account-management/authentication-and-permissions/access-policies/ |
| Alloy remotecfg block reference | https://grafana.com/docs/alloy/latest/reference/config-blocks/remotecfg/ |

---

*This document is maintained by the Grafana Cloud observability team. For questions, open an issue or contact your Grafana support representative.*
