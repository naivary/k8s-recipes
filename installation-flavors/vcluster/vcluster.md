# vCluster

## Enterprise Features only

Some of the features on vCluster are enterprise only. The question remaining is
can you still implement thgose features manually and they just selling the
automation?

- Cert-Manager
- External Secrets Operator
- Istio
- KubeVirt

## Free Features

- Metrics Server

## Installation

The installation is mostly done using the helm chart. Therefore, it is possible
to have a complete offline installation when the helm chart is hosted in a
private registry which is accessible from the cluster.

Furthermore, the installation does NOT require cluster-admin priviliges. As long
as the user is allowed to deploy `StatefulSets` or `Deployments` a vCluster can
be installed.

In the background vCluster is using helm to install the cluster. Any
configuration definde in a `vcluster.yaml` is mapped 1:1 to the `values.yaml` of
the [helm chart](https://artifacthub.io/packages/helm/loft/vcluster).

## GitOps

Currently the documentation is only stating how to install a central `FluxCD`
for an GitOps approach. ArgoCD is not documented.

## Storage

vCluster is using SQLite as an embedded storage. That is acceptable if loosing
the data on restart is not crucial (e.g. test environments). An external storage
(MySQL, PostgreSQL) can be defined but that is currently an enterprise feature.
etcd is included in the free-tier.

## Nodes

when running `k get nodes` in a vCluster it only reorts the nodes on which the
[vCluster pods](https://www.vcluster.com/docs/vcluster/deploy/worker-nodes/host-nodes)
are running. The actual nodes which are available for scheduling are not
reported.

## ArgoCD

Because vCluster is just a helm chart in can be deployed using ArgoCD. For that
it is required to use the official
[helm chart](https://artifacthub.io/packages/helm/loft/vcluster) with the wished
version. An example ArgoCD Application can look something like this:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vcluster-ha
  namespace: argocd
spec:
  project: default
  syncPolicy:
    automated:
      enabled: true
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
  source:
    repoURL: https://charts.loft.sh
    targetRevision: 0.32.0-next.0
    chart: vcluster
    helm:
      releaseName: vcluster-team-01
      parameters:
        - name: "controlPlane.backingStore.etcd.deploy.enabled"
          value: "true"
        - name: "controlPlane.backingStore.etcd.deploy.statefulSet.highAvailability.replicas"
          value: "3"
        - name: "controlPlane.statefulSet.highAvailability.replicas"
          value: "3"
  destination:
    server: "https://kubernetes.default.svc"
    namespace: team-01
```
