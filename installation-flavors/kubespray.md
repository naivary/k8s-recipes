# Minimal Configuration Principle

My recommendation for the configuration of the cluster is to keep it as minimal
as possible. The installation of Kubernetes Apps which are defined in
`addons.yaml` can hardly be managed using GitOps pratices because you don't
control the installation method (helm, kustomize, native manifests etc.)

Therefore I recommened the following configuration for the cluster:

```yaml
# k8s-cluster.yaml
#
# This is not officially defined in the k8s-cluster.yaml but used
# internally to check if the instalation of the kube-proxy is required.
# Because we are using Cilium in kube-proxy replacement mode the native
# kube-proxy component is no longer required.
kube_proxy_remove: true

# Don't use cilium here and install it using helm and your custom values.yaml
# to be able to control the installation and lifecycle using ArgoCD.
kube_network_plugin: cni

# avoid permission bugs of cilium
kube_owner: root

# free lunch / nice to haves
kubernetes_audit: true
remove_anonymous_access: true
kube_encrypt_secret_data: true
```

# Post-Installation Steps

After you have setup your Kubernetes cluster you need to install the CNI for the
cluster to be funtctioning correctly. The recommened CNI is Cilium because it's
open-source, supported by the CNCF and battle tested by major companies like
Google and Microsoft.

Install Cilium using the official helm chart and a custom `values.yaml`. If you
don't have specific needs an example `values.yaml` can be found under
[here](../cilium/values.yaml).

If you plan to use the Gateway API as your main way for public traffic you have
to install the gateway API CRDs for your used Cilium version. The compatability
of the versions can be found
[here](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/).
