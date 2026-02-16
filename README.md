# ECK Deployment Examples for ArgoCD

Deployment artifacts for [Elastic Cloud on Kubernetes (ECK)](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/install-using-helm-chart) to be deployed via **ArgoCD**. The ECK operator is installed from the official Elastic Helm chart; Elasticsearch and Kibana are deployed using the official **eck-stack** Helm chart with custom value files.

## Structure

```
.
├── README.md
├── apps/
│   ├── application-operator.yaml   # ArgoCD Application for ECK operator (Helm)
│   ├── application-single-node.yaml
│   └── application-multi-node.yaml
└── clusters/
    └── eck-stack/                # Single chart, shared values + profile overrides
        ├── Chart.yaml            # Wrapper chart (depends on eck-stack)
        ├── values.yaml           # Shared config (versions, Kibana, disabled components)
        ├── values-single-node.yaml   # Overrides: 1 ES node, 30Gi
        └── values-multi-node.yaml   # Overrides: 2 ES nodes, 50Gi
```

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- Helm 3.2+ (for local `helm dependency update` if you change chart versions)

## Installation

### 1. Bootstrap the ECK operator

Apply the operator Application so ArgoCD installs the ECK operator from the official Helm repo:

```bash
kubectl apply -f apps/application-operator.yaml
```

Wait until the `eck-operator` Application is **Synced** and **Healthy**. The operator runs in the `elastic-system` namespace.

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

Then apply them. ArgoCD will render the Helm chart from `clusters/eck-stack` (which pulls in the official **eck-stack** chart as a dependency), using `values.yaml` plus either `values-single-node.yaml` or `values-multi-node.yaml` (merged in that order).

### 3. Helm dependencies (when using Git as source)

The cluster charts depend on the **eck-stack** chart from `https://helm.elastic.co`. ArgoCD can resolve Helm dependencies from the chart’s `Chart.yaml` when the repo is a Helm-capable source. If your ArgoCD setup does not run `helm dependency build`, either:

- Use a CI step that runs `helm dependency build` in `clusters/eck-stack` and commits `Chart.lock` and `charts/*.tgz`, or  
- Configure ArgoCD to use a Helm repository source for **eck-stack** and your Git repo only for the wrapper chart and values (e.g. via ApplicationSet or a chart that references the OCI/Helm repo).

To generate `Chart.lock` and the `charts/` directory locally:

```bash
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

## Customization

- **Stack version**: In `clusters/eck-stack/values.yaml`, set `eck-elasticsearch.version` and `eck-kibana.version` (e.g. `9.3.0`). Keep them aligned.
- **Storage**: Adjust `volumeClaimTemplates` in `values.yaml` (default) or in the profile files (per-environment overrides).
- **Node count / profile**: `values-single-node.yaml` and `values-multi-node.yaml` override only the Elasticsearch `nodeSets` (count and storage); edit them to add node sets or change counts.
- **Operator**: Chart version is set in the operator Application (`spec.source.targetRevision`). Update it to match the [ECK Helm chart](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/install-using-helm-chart) version you want.

## References

- [Install ECK using a Helm chart](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/install-using-helm-chart)
- [Manage deployments (ECK)](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/manage-deployments)
- [ECK Stack Helm chart (Artifact Hub)](https://artifacthub.io/packages/helm/elastic/eck-stack)
