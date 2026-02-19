# ECK Deployment Examples for ArgoCD

Deployment artifacts for [Elastic Cloud on Kubernetes (ECK)](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/install-using-helm-chart) to be deployed via **ArgoCD**. The ECK operator is installed via a wrapper chart (official **eck-operator** + optional trial license); Elasticsearch and Kibana are deployed using the official **eck-stack** Helm chart with custom value files.

## Structure

```
.
├── README.md
├── apps/
│   ├── application-operator.yaml       # ArgoCD Application for ECK operator (wrapper chart)
│   ├── application-single-node.yaml
│   ├── application-multi-node.yaml
│   └── application-stack-monitoring.yaml
├── operator/
│   └── eck-operator/             # Wrapper around official eck-operator chart
│       ├── Chart.yaml            # Depends on elastic/eck-operator
│       ├── values.yaml           # trialLicense.enabled, subchart overrides
│       ├── values-trial.yaml     # Optional: enable enterprise trial license
│       └── templates/
│           └── eck-trial-license-secret.yaml   # Created when trialLicense.enabled
└── clusters/
    ├── eck-stack/                # Single chart, shared values + profile overrides
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   ├── values-single-node.yaml
    │   └── values-multi-node.yaml
    └── stack-monitoring/         # Raw ECK CRs (elastic1, elastic2, elastic-monitoring)
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- Helm 3.2+ (for local `helm dependency update` if you change chart versions)

## Installation

### 1. Bootstrap the ECK operator

Set `spec.source.repoURL` and `spec.source.targetRevision` in `apps/application-operator.yaml` to your Git repo, then apply:

```bash
kubectl apply -f apps/application-operator.yaml
```

ArgoCD will sync the **operator/eck-operator** chart (which wraps the official eck-operator Helm chart) into `elastic-system`. Wait until the Application is **Synced** and **Healthy**.

**Enterprise trial license:** To enable the trial license, either set `trialLicense.enabled: true` in `operator/eck-operator/values.yaml`, or add `values-trial.yaml` to the Application's `helm.valueFiles`. When enabled, the chart creates the `eck-trial-license` Secret in `elastic-system` (label `license.k8s.elastic.co/type: enterprise_trial`, annotation `elastic.co/eula: accepted`).

### 2. Point cluster Applications to your Git repo

Edit the cluster Applications and set `spec.source.repoURL` and `spec.source.targetRevision` to your Git repository:

```bash
# Replace with your repo URL and branch/tag
kubectl apply -f apps/application-single-node.yaml
kubectl apply -f apps/application-multi-node.yaml
```

Or clone this repo into your Git server and update the `repoURL` in:

- `apps/application-single-node.yaml`
- `apps/application-multi-node.yaml`
- `apps/application-stack-monitoring.yaml` (optional; see [Stack monitoring](#stack-monitoring))

Then apply them. ArgoCD will render the Helm chart from `clusters/eck-stack` (which pulls in the official **eck-stack** chart as a dependency), using `values.yaml` plus either `values-single-node.yaml` or `values-multi-node.yaml` (merged in that order).

### 3. Helm dependencies (when using Git as source)

The **operator/eck-operator** and **clusters/eck-stack** charts depend on charts from `https://helm.elastic.co`. ArgoCD can resolve Helm dependencies from the chart’s `Chart.yaml` when the repo is a Helm-capable source. If your ArgoCD setup does not run `helm dependency build`, either:

- Use a CI step that runs `helm dependency build` in `clusters/eck-stack` and commits `Chart.lock` and `charts/*.tgz`, or  
- Configure ArgoCD to use a Helm repository source for **eck-stack** and your Git repo only for the wrapper chart and values (e.g. via ApplicationSet or a chart that references the OCI/Helm repo).

To generate `Chart.lock` and the `charts/` directory locally:

```bash
cd operator/eck-operator && helm dependency update && cd ../..
cd clusters/eck-stack && helm dependency update && cd ../..
```

You can commit the resulting `Chart.lock` and `charts/` so ArgoCD has everything in Git.

## Cluster examples

| Example      | Elasticsearch nodes | Kibana | Namespace |
|-------------|---------------------|--------|-----------|
| single-node | 1                   | 1      | `elastic` |
| multi-node  | 2                   | 1      | `elastic` |

- **single-node**: minimal setup (e.g. dev/test); one Elasticsearch node and one Kibana replica.
- **multi-node**: two Elasticsearch nodes for better availability; one Kibana replica.

To use different namespaces, change `spec.destination.namespace` in the corresponding Application manifest.

## Stack monitoring

The **stack-monitoring** Application deploys a dedicated [Stack Monitoring](https://www.elastic.co/docs/deploy-manage/monitor/stack-monitoring/eck-stack-monitoring) setup:

| Namespace           | Resources        | Role |
|---------------------|------------------|------|
| `elastic1`          | 1 Elasticsearch  | Monitored cluster (metrics/logs → monitoring) |
| `elastic2`          | 1 Elasticsearch  | Monitored cluster (metrics/logs → monitoring) |
| `elastic-monitoring`| 1 Elasticsearch + 1 Kibana | Monitoring cluster; Stack Monitoring UI and **CCS** (Cross-Cluster Search) to elastic1/elastic2 |

The two monitored clusters have **remote cluster server** enabled; the monitoring cluster is configured with **remote clusters** (API key) so you can run [Cross-Cluster Search (CCS)](https://www.elastic.co/docs/deploy-manage/remote-clusters/eck-remote-clusters) from the monitoring Kibana against `elastic1` and `elastic2`.

For clusters that enforce NetworkPolicies (e.g. default-deny), the stack-monitoring chart can generate policies so that metrics/logs and CCS traffic are allowed between namespaces. See [NetworkPolicies](network-policies.md) for a description and diagrams of the deployed policies and why they are needed.

Deploy after the ECK operator is running:

```bash
kubectl apply -f apps/application-stack-monitoring.yaml
```

Set `spec.source.repoURL` and `spec.source.targetRevision` to your Git repo. The chart renders ECK Elasticsearch and Kibana CRs with explicit namespaces; Metricbeat and Filebeat sidecars are added by ECK for stack monitoring.

**Cross-namespace RBAC:** If the ECK operator is started with `--enforce-rbac-on-refs` (e.g. operator with `values-rbac.yaml`), set `enforceRBAC: true` in `clusters/stack-monitoring/values.yaml` or add `values-enforce-rbac.yaml` to the stack-monitoring Application's `valueFiles`. The chart will then create the [ClusterRole, ServiceAccounts, and RoleBindings](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/restrict-cross-namespace-resource-associations) so the monitored clusters (elastic1, elastic2) can associate with the monitoring Elasticsearch in `elastic-monitoring`.

Access Kibana for the monitoring cluster with:

```bash
kubectl -n elastic-monitoring port-forward svc/kibana-kb-http 5601:5601
```

## Customization

- **Stack version**: In `clusters/eck-stack/values.yaml`, set `eck-elasticsearch.version` and `eck-kibana.version` (e.g. `9.3.0`). Keep them aligned.
- **Storage**: Adjust `volumeClaimTemplates` in `values.yaml` (default) or in the profile files (per-environment overrides).
- **Node count / profile**: `values-single-node.yaml` and `values-multi-node.yaml` override only the Elasticsearch `nodeSets` (count and storage); edit them to add node sets or change counts.
- **Operator**: ECK operator version is set in `operator/eck-operator/Chart.yaml` (dependency version). Update it to match the [ECK Helm chart](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/install-using-helm-chart) version you want.
- **Trial license**: Set `trialLicense.enabled: true` in `operator/eck-operator/values.yaml` or use `values-trial.yaml` in the operator Application's `valueFiles` to create the `eck-trial-license` Secret. By setting this to true you are expressing that you have accepted the Elastic EULA which can be found at https://www.elastic.co/eula.

## Accessing the Kibana instance

- Port forwarding to access Kibana in your browser:

```bash
kubectl -n elastic port-forward svc/kibana-kb-http 5601:5601
```

- Retrieve the password for default user 'elastic':

```bash
kubectl -n elastic get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```


- For stack monitoring example use:

```bash
kubectl -n elastic-monitoring port-forward svc/kibana-kb-http 5601:5601
```

```bash
kubectl -n elastic-monitoring get secret monitoring-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```
 

## References

- [NetworkPolicies](network-policies.md) — Stack monitoring NetworkPolicies (flow diagrams, ports, and rationale)
- [Install ECK using a Helm chart](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/install-using-helm-chart)
- [Manage deployments (ECK)](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/manage-deployments)
- [Enable stack monitoring on ECK](https://www.elastic.co/docs/deploy-manage/monitor/stack-monitoring/eck-stack-monitoring)
- [Restrict cross-namespace resource associations (RBAC)](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/restrict-cross-namespace-resource-associations)
- [Remote clusters on ECK (CCS)](https://www.elastic.co/docs/deploy-manage/remote-clusters/eck-remote-clusters)
- [ECK Stack Helm chart (Artifact Hub)](https://artifacthub.io/packages/helm/elastic/eck-stack)
