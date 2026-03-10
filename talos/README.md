# Create a Talos Linux Cluster on VirtualBox

## Download ISO Image

Install a correctly configured ISO Image for your targeted platform from the
official Talos [Linux Image Factory](https://factory.talos.dev/).

## Create virtual machines using VirtualBox

Follow the
[official](https://docs.siderolabs.com/talos/v1.12/platform-specific-installations/local-platforms/virtualbox)
documentation and outlined steps to produce virtual machines using the just
downloaded ISO image.

Some missing information on the official documentation are:

1. Choose `Linux` as `Type`
1. Choose `Other Linux` as `Subtype`
1. Choose `Other Linux (64-bit)` as `Version`
1. Choose the just downloaded ISO Image as the booting image.
1. Choose the sizing by consulting the [official] documentation for the hardware
   requirements
1. Make sure to configure two network adapters. The first one should be a
   `bridge` and the second a `host-only` adapter.
1. The last step on the official documentation can be skipped.

## Configure and Bootstrap the Control Plane

After starting up the just created virtual machine you will see many information
regarding the current state of the machine. When the machine is finally in
maintenance mode the booting process can be triggered.

To trigger the booting process the following commands should be executed on a
system which is meeting the following requirements:

1. Linux OS
1. Can reach the control plane over a common network interface (e.g. host-only
   adapter)
1. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
1. [talosctl](https://docs.siderolabs.com/talos/v1.12/getting-started/talosctl)

```bash
export OUT_DIR=_out
export TALOSCONFIG="$OUT_DIR/talosconfig"
export KUBECONFIG="$OUT_DIR/kubeconfig"
export CONTROL_PLANE_IP=192.168.X.X
```

```bash
talosctl gen config talos-vbox-cluster https://$CONTROL_PLANE_IP:6443 --output-dir $OUT_DIR

talosctl --talosconfig $TALOSCONFIG config endpoint $CONTROL_PLANE_IP
talosctl --talosconfig $TALOSCONFIG config node $CONTROL_PLANE_IP
```

The default CNI of Talos Linux is `flannel`. If you wish to install Cilium or
any other CNI some custom configuration work is needed in the
`controlplane.yaml`

```yaml

...
cluster:
  cni:
    name: none
  proxy:
    disabled: true
...
```

For all the configuration options available consult the reference section of the
[official documentation](https://docs.siderolabs.com/talos/v1.12/reference/configuration/v1alpha1/config).
It is considered best-pratice to use
[patches](https://docs.siderolabs.com/talos/v1.12/configure-your-talos-cluster/system-configuration/patching)
instead of changing configuration options directly in the generated YAML files.

After finishing the configuration process of your desired state, the
configuration can be applied to the control plane and etcd can be bootstrapped.

```bash
talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP --file "$OUT_DIR/controlplane.yaml"
```

This step can be repeated multiple times to configure N control planes. Make
sure to use the correct IP-Address.

```bash
talosctl --talosconfig $TALOSCONFIG bootstrap
```

## Configure and Bootstrap Woker Nodes

```bash
talosctl apply-config --insecure --nodes $WORKER_IP --file "$OUT_DIR/worker.yaml"
```

This step can be repeated multiple times to configure N worker nodes.

## Retrieve a Kubeconfig

```bash
talosctl --talosconfig $TALOSCONFIG kubeconfig $OUT_DIR
```
