## Operating System Requirements

```
$ lsb_release -a
```

```
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.1 LTS
Release:        24.04
Codename:       noble
```

## Kernel Version

```
$ uname -a
```

```
Linux k8s-n1 6.8.0-41-generic #41-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug  2 20:41:06 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

## Prepare the operating system (All Nodes)

```bash
# start an interactive root shell
sudo -i
```

```bash
# udpate the operating system packages
apt-get update && apt-get upgrade -y
```

```bash
# install required packages
apt-get install -y apt-transport-https ca-certificates curl
```

```bash
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

```bash
swapoff -a
```

### Networking (All Nodes)

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
modprobe overlay
modprobe br_netfilter
```

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

```bash
sysctl --system
```

## Install containerd (All Nodes)

```bash
tee /etc/modules-load.d/containerd.conf << EOF
overlay
br_netfilter
EOF
```

```bash
sysctl --system
```

```bash
apt install curl gnupg2 software-properties-common apt-transport-https ca-certificates -y
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
```

```bash
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

```bash
apt update
apt install containerd.io -y
```

```bash
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml
```

```bash
vim /etc/containerd/config.toml

# ...
# [plugins."io.containerd.grpc.v1.cri"]
#   ...
#   sandbox_image = "registry.k8s.io/pause:3.10"
#   ...
# [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
#   ...
#   SystemdCgroup = true
#   ...
```

```bash
systemctl restart containerd
systemctl enable containerd
```

## Configure Kubernetes Controlplane (control plane only)

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  # !TODO! Change this to your correct Address
  advertiseAddress: "192.168.56.61"
  bindPort: 6443
skipPhases:
  - addon/kube-proxy
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
apiServer:
  certSANs:
    - "127.0.0.1"
    - "192.168.56.61"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

```bash
kubeadm init --config kubeadm-config.yaml
```

## Join Worker Node

```bash
kubeadm join <...>
```

```bash
# check status of nodes with kubectl
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes

# The nodes are not ready because no CNI is installed!!
# NAME     STATUS     ROLES           AGE   VERSION
# k8s-m1   NotReady   control-plane   37s   v1.33.1
# k8s-n1   NotReady   <none>          19s   v1.33.1
```

## Install Cilium (CNI) (Control Plane)

```bash
# Run on control plane
# Delete the configmap as well to avoid kube-proxy being reinstalled during a Kubeadm upgrade (works only for K8s 1.19 and newer)
kubectl -n kube-system delete ds kube-proxy
# Run on each node with root permissions:
kubectl -n kube-system delete cm kube-proxy
```

```bash
# Run on each node with root permissions
iptables-save | grep -v KUBE | iptables-restore
```


```bash
# Install helm (kubernetes package manager)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

```bash
# add cilium helm repo
helm repo add cilium https://helm.cilium.io/
```

```bash
# !TODO! Change the IP to the `advertiseAddress` of the kubeadm config file
API_SERVER_IP=192.168.56.61
API_SERVER_PORT=6443
helm install cilium cilium/cilium --version 1.17.4 \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=${API_SERVER_IP} \
    --set k8sServicePort=${API_SERVER_PORT}
```

## Installation Verification

```bash
kubectl get nodes

# After installing the CNI (Cilium) nodes are ready for use
#
# NAME     STATUS   ROLES           AGE     VERSION
# k8s-m1   Ready    control-plane   3m57s   v1.33.1
# k8s-n1   Ready    <none>          3m39s   v1.33.1
```
