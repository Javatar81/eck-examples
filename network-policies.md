# NetworkPolicies in Stack Monitoring

This document describes the **NetworkPolicies** deployed by the **stack-monitoring** chart and why they are needed when running Stack Monitoring with Cross-Cluster Search (CCS) in a locked-down Kubernetes environment.

## Overview

The stack-monitoring setup uses **three namespaces**:

| Namespace           | Role                | Components                                      |
|---------------------|---------------------|-------------------------------------------------|
| `elastic1`          | Monitored cluster   | 1 Elasticsearch (metrics/logs → monitoring)     |
| `elastic2`          | Monitored cluster   | 1 Elasticsearch (metrics/logs → monitoring)     |
| `elastic-monitoring`| Monitoring cluster  | 1 Elasticsearch + 1 Kibana (CCS, Stack UI)     |

When **NetworkPolicy** is enforced in the cluster (e.g. by a default-deny policy or a CNI that enforces policies), pods can only talk to each other if explicit policies allow it. The stack-monitoring chart can generate two sets of NetworkPolicies so that:

1. **Stack monitoring** (Metricbeat/Filebeat) can push metrics and logs from monitored clusters to the monitoring cluster.
2. **Cross-Cluster Search (CCS)** can work: the monitoring cluster’s Elasticsearch can query the remote cluster servers in `elastic1` and `elastic2`.

Enable them in `clusters/stack-monitoring/values.yaml` (or via extra value files):

- `generateStackMonitoringPolicies: true` — for metrics/logs (port 9200).
- `generateRenoteClustersSearchPolicies: true` — for CCS (port 9443).

---

## Why We Need Them

- **Default-deny or strict clusters:** If the cluster or namespaces use a default-deny NetworkPolicy, all pod-to-pod traffic is blocked unless allowed by a policy. Without these policies, Metricbeat cannot reach the monitoring Elasticsearch, and the monitoring cluster cannot reach the remote cluster server ports in `elastic1`/`elastic2`.
- **Least privilege:** Even without default-deny, these policies restrict traffic to the minimum required: only the right namespaces and ports (9200 for monitoring, 9443 for CCS) are allowed, which reduces blast radius and meets security best practices.
- **Explicit documentation:** The policies make the required network paths (who talks to whom and on which port) explicit and version-controlled.

---

## 1. Stack Monitoring Policies (port 9200)

**Purpose:** Allow Metricbeat/Filebeat sidecars in the **monitored** Elasticsearch pods to send metrics and logs to the **monitoring** cluster’s Elasticsearch (HTTP API on port 9200).

**Enabled with:** `generateStackMonitoringPolicies: true`

### Policies

| Policy name                      | Namespace         | Type   | Effect |
|----------------------------------|-------------------|--------|--------|
| `allow-monitored-clusters-ingress` | `elastic-monitoring` | Ingress | Allow traffic **from** `elastic1` and `elastic2` **to** monitoring Elasticsearch on **9200** |
| `allow-egress-to-monitoring-cluster` | `elastic1`, `elastic2` | Egress  | Allow traffic **from** monitored Elasticsearch pods **to** `elastic-monitoring` on **9200**, plus DNS (UDP 53) |

### Flow (stack monitoring)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  MONITORED NAMESPACES (elastic1, elastic2)                                       │
│  ┌──────────────────────────────────┐  ┌──────────────────────────────────┐     │
│  │ elastic1                         │  │ elastic2                         │     │
│  │  ┌─────────────────────────────┐ │  │  ┌─────────────────────────────┐ │     │
│  │  │ Elasticsearch + Metricbeat  │ │  │  │ Elasticsearch + Metricbeat  │ │     │
│  │  │ sidecar                     │ │  │  │ sidecar                     │ │     │
│  │  └──────────────┬──────────────┘ │  │  └──────────────┬──────────────┘ │     │
│  └─────────────────┼────────────────┘  └─────────────────┼────────────────┘     │
│                     │  egress 9200 + DNS 53                │  egress 9200 + DNS  │
│                     │  (allow-egress-to-monitoring-cluster)                      │
└─────────────────────┼─────────────────────────────────────┼─────────────────────┘
                      │                                     │
                      ▼                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  MONITORING NAMESPACE (elastic-monitoring)                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  Monitoring Elasticsearch (port 9200)                                    │   │
│  │  ingress from elastic1, elastic2 (allow-monitored-clusters-ingress)      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Remote Cluster Search (CCS) Policies (port 9443)

**Purpose:** Allow the **monitoring** cluster’s Elasticsearch to connect to the **remote cluster server** in each monitored cluster (port 9443, default for remote cluster with API key auth) so that Cross-Cluster Search from the monitoring Kibana works.

**Enabled with:** `generateRenoteClustersSearchPolicies: true`

### Policies

| Policy name                         | Namespace         | Type   | Effect |
|-------------------------------------|-------------------|--------|--------|
| `allow-egress-to-monitored-clusters-ccs` | `elastic-monitoring` | Egress  | Allow traffic **from** monitoring Elasticsearch **to** `elastic1` and `elastic2` on **9443**, plus DNS (UDP 53) |
| `allow-monitoring-cluster-ingress-ccs`    | `elastic1`, `elastic2` | Ingress | Allow traffic **from** `elastic-monitoring` **to** monitored Elasticsearch on **9443** |

### Flow (cross-cluster search)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  MONITORING NAMESPACE (elastic-monitoring)                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  Monitoring Elasticsearch                                                │   │
│  │  egress to elastic1:9443, elastic2:9443 (allow-egress-to-monitored-...)  │   │
│  └──────────────────────────────────┬──────────────────────────────────────┘   │
└─────────────────────────────────────┼──────────────────────────────────────────┘
                                      │
         egress 9443 + DNS 53         │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  MONITORED NAMESPACES (elastic1, elastic2)                                       │
│  ┌──────────────────────────────────┐  ┌──────────────────────────────────┐     │
│  │ elastic1                         │  │ elastic2                         │     │
│  │  ┌─────────────────────────────┐ │  │  ┌─────────────────────────────┐ │     │
│  │  │ Elasticsearch               │ │  │  │ Elasticsearch               │ │     │
│  │  │ remote cluster server :9443 │ │  │  │ remote cluster server :9443 │ │     │
│  │  │ ingress from elastic-        │ │  │  │ ingress from elastic-        │ │     │
│  │  │ monitoring (allow-...-ccs)   │ │  │  │ monitoring (allow-...-ccs)  │ │     │
│  │  └─────────────────────────────┘ │  │  └─────────────────────────────┘ │     │
│  └──────────────────────────────────┘  └──────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Combined View (both policy sets enabled)

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │ elastic-monitoring                                           │
                    │  • Monitoring ES (9200, 9443 client)                          │
                    │  • Kibana (Stack Monitoring UI, CCS)                          │
                    └───┬─────────────────────────────────────────────┬───────────┘
                        │                                             │
        ingress 9200    │         egress 9443 (+ DNS)                  │
        (metrics/logs)  │         (CCS to remote clusters)             │
                        │                                             │
    ┌───────────────────┼───────────────────┐     ┌──────────────────┼───────────┐
    │ elastic1          │                   │     │ elastic2          │           │
    │  ES + Metricbeat  │◄──────────────────┼─────┼► ES + Metricbeat  │           │
    │  remote cluster   │   ingress 9443    │     │  remote cluster   │           │
    │  server :9443     │                   │     │  server :9443     │           │
    │  egress → monitoring:9200 (+ DNS)     │     │  egress → monitoring:9200     │
    └───────────────────┴───────────────────┘     └──────────────────┴───────────┘
```

---

## Summary

| Direction                    | Port | Purpose                    | Enabled by                            |
|-----------------------------|------|----------------------------|----------------------------------------|
| Monitored → Monitoring      | 9200 | Metricbeat/Filebeat push   | `generateStackMonitoringPolicies`     |
| Monitoring → Monitored      | 9443 | Cross-Cluster Search (CCS) | `generateRenoteClustersSearchPolicies` |

Both policy sets also allow **DNS (UDP 53)** where needed so that services in other namespaces can be resolved. Without these NetworkPolicies, stack monitoring and CCS will fail in environments that enforce network isolation (e.g. default-deny or strict CNI policies).
