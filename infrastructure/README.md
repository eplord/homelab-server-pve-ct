# Infrastructure

This directory houses the code that transforms raw bare-metal machines into functional Kubernetes clusters. The code in this directory depends on a LAN-accessible Proxmox VE installation (or multiple!) to create a bare-bones Kubernetes cluster using [Talos](https://github.com/siderolabs/talos).

## Prerequisites

1. Terraform installed (check [the providers file](./talos/providers.tf) for the specific version requirements)
2. Install Proxmox VE v8.0+ on a baremetal machine (or more than one).
3. Ensure you have a file named `aws-credentials` in the `talos` directory in the format:

    ```text
    [default]
    aws_access_key_id = <redacted>
    aws_secret_access_key = <redacted>
    ```

    This provides authentication to the AWS S3 bucket backend that stores the TF state

## How To Use

1. Populate `main.tf` as desired.
2. Run the desired Terraform commands (i.e. `terraform plan`, `terraform apply`).
3. Once the Kubernetes cluster is successfully applied, run this command to get the necessary configs:

    ```bash
    source get_configs
    ```

## Gotchas

1. I currently use a forked version of the Unifi Terraform provider which fixes a missing IP bug. PR is [here](https://github.com/paultyng/terraform-provider-unifi/pull/430).
2. For some reason the certs aren't correct on the initial cluster installation. Commands such as `kubectl logs` don't work as a result (but the cluster still functions as normal). To fix this, I needed to run the commands found in [this GitHub comment](https://github.com/kubernetes/kubeadm/issues/591#issuecomment-1257061416).

   I need to run the approval command for each new node in the cluster or else I get some weird warnings, can't view logs, the status of control-plane component is unknown, and more. This problem could probably be debugged and automated

## Operations

### To add new nodes with new disks

1. Create LVM-Thin storage pool for the new disk
2. Assign new nodes to that disk

### Upgrading Kubernetes

Can perform a rolling upgrade with the TF provider and a standard TF workflow (or `talosctl` also works per [the docs](https://www.talos.dev/v1.6/kubernetes-guides/upgrading-kubernetes/)).

### Upgrade Talos

The Talos TF provider is relatively under-featured for upgrades. [Example](https://github.com/siderolabs/terraform-provider-talos/issues/140#issue-2055027252). So it's best to use `talosctl` and follow the more production-ready upgrade path [here](https://www.talos.dev/v1.6/talos-guides/upgrading-talos/).

### Upgrading Talos-Controlled, Kubernetes-State-Managed Resources

There are some Kubernetes configurations, such as the `kube-proxy` configuration, which `talosctl` manages but only touches during Kubernetes bootstraps/upgrades. [Here](https://github.com/siderolabs/talos/discussions/7835) is a good example. To update a resource whose state is entirely within Kubernetes, but the config is managed via Talos, refer to the [upgrading Kubernetes section](#upgrading-kubernetes) above

## TODOs

1. Setup [Talos virtual IP](https://www.talos.dev/v1.6/talos-guides/network/vip/) correctly so that I can access the Kubernetes API from any node (not just one master node, as I have it now).

## Rebuild Cluster

As I'm beginning to get a handle on what this cluster should look like, I want to pause and reset the cluster before putting any of my important applications inside of it. This list is the stuff that should be taken care of before important apps live on it:

1. Changing the VLAN so that Kubernetes is inside 192.168.2.x instead of the same 192.168.1.x as the rest of my home network.
2. Tearing out Flanel for Cilium.
