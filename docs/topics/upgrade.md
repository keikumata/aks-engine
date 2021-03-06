# Upgrading Kubernetes Clusters

## Prerequisites

All the commands in this guide require both the Azure CLI and `aks-engine`. Follow the [quickstart guide](../tutorials/quickstart.md) before continuing.

This guide assumes you already have deployed a cluster using `aks-engine`. For more details on how to do that see [deploy](../tutorials/deploy.md).

## Upgrade

This document provides guidance on how to upgrade the Kubernetes version for an existing AKS Engine cluster and recommendations for adopting `aks-engine upgrade` as a tool.

<a name="pre-requirements"></a>

### Know before you go

In order to ensure that your `aks-engine upgrade` operation runs smoothly, there are a few things you should be aware of before getting started.

1) You will need access to the `apimodel.json` that was generated by `aks-engine deploy` or `aks-engine generate` (by default this file is placed into a relative directory that looks like `_output/<clustername>/`). `aks-engine` will use the `--api-model` argument to introspect the `apimodel.json` file in order to determine the cluster's current Kubernetes version, as well as all other cluster configuration data as defined by `aks-engine` during the last time that `aks-engine` was used to deploy, scale, or upgrade the cluster.

2) `aks-engine upgrade` expects a cluster configuration that conforms to the current state of the cluster. In other words, the Azure resources inside the resource group deployed by `aks-engine` should be in the same state as when they were originally created by `aks-engine`. If you perform manual operations on your Azure IaaS resources (other than `aks-engine scale` and `aks-engine upgrade`) DO NOT use `aks-engine upgrade`, as the aks-engine-generated ARM template won't be reconcilable against the state of the Azure resources that reside in the resource group. This includes naming of resources; `aks-engine upgrade` relies on some resources (such as VMs) to be named in accordance with the original `aks-engine` deployment. In summary, the set of Azure resources in the resource group are mutually reconcilable by `aks-engine upgrade` only if they have been exclusively created and managed as the result of a series of successive ARM template deployments originating from `aks-engine`.

3) `aks-engine upgrade` allows upgrading the Kubernetes version to any AKS Engine-supported patch release in the current minor release channel that is greater than the current version on the cluster (e.g., from `1.16.4` to `1.16.6`), or to the next aks-engine-supported minor version (e.g., from `1.16.6` to `1.17.2`). (Or, see [`aks-engine upgrade --force`](#force-upgrade) if you want to bypass AKS Engine "supported version requirements"). In practice, the next AKS Engine-supported minor version will commonly be a single minor version ahead of the current cluster version. However, if the cluster has not been upgraded in a significant amount of time, the "next" minor version may have actually been deprecated by aks-engine. In such a case, your long-lived cluster will be upgradable to the nearest, supported minor version that `aks-engine` supports at the time of upgrade (e.g., from `1.11.10` to `1.13.11`).

    To get the list of all available Kubernetes versions and upgrades, run the `get-versions` command:

    ```bash
    ./bin/aks-engine get-versions
    ```

    To get the versions of Kubernetes that your particular cluster version is upgradable to, provide its current Kubernetes version in the `version` arg:

    ```bash
    ./bin/aks-engine get-versions --version 1.12.8
    ```

4) `aks-engine upgrade` relies upon a working connection to the cluster control plane during upgrade, both (1) to validate successful upgrade progress, and (2) to cordon and drain nodes before upgrading them, in order to minimize operational downtime of any running cluster workloads. If you are upgrading a **private cluster**, you must run `aks-engine upgrade` from a host VM that has network access to the control plane, for example a jumpbox VM that resides in the same VNET as the master VMs. For more information on private clusters [refer to this documentation](features.md#feat-private-cluster).

5) If using `aks-engine upgrade` in production, it is recommended to stage an upgrade test on an cluster that was built to the same specifications (built with the same cluster configuration + the same version of the `aks-engine` binary) as your production cluster before performing the upgrade, especially if the cluster configuration is "interesting", or in other words differs significantly from defaults. The reason for this is that AKS Engine supports many different cluster configurations and the extent of E2E testing that the AKS Engine team runs cannot practically cover every possible configuration. Therefore, it is recommended that you ensure in a staging environment that your specific cluster configuration is upgradable using `aks-engine upgrade` before attempting this potentially destructive operation on your production cluster.

6) `aks-engine upgrade` is backwards compatible. If you deployed with `aks-engine` version `0.27.x`, you can run upgrade with version `0.29.y`. In fact, it is recommended that you use the latest available `aks-engine` version when running an upgrade operation. This will ensure that you get the latest available software and bug fixes in your upgraded cluster.

7) `aks-engine upgrade` will automatically re-generate your cluster configuration to best pair with the desired new version of Kubernetes, and/or the version of AKS Engine that is used to execute `aks-engine upgrade`. To use an example of both:

- When you upgrade to (for example) Kubernetes 1.14 from 1.13, AKS Engine will automatically change your control plane configuration (e.g., `coredns`, `metrics-server`, `kube-proxy`) so that the cluster component configurations have a close, known-working affinity with 1.14.
- When you perform an upgrade, even if it is a Kubernetes patch release upgrade such as 1.14.1 to 1.14.2, but you use a newer version of AKS Engine, a newer version of `etcd` (for example) may have been validated and configured as default since the original version of AKS Engine used to build the cluster was released. So, for example, without any explicit user direction, the newly upgraded cluster will now be running etcd v3.2.26 instead of v3.2.25. _This is by design._

In summary, using `aks-engine upgrade` means you will freshen and re-pave the entire stack that underlies Kubernetes to reflect the best-known, recent implementation of Azure IaaS + OS + OS config + Kubernetes config.

### Under the hood

During the upgrade, *aks-engine* successively visits virtual machines that constitute the cluster (first the master nodes, then the agent nodes) and performs the following operations:

Master nodes:

- cordon the node and drain existing workloads
- delete the VM
- create new VM and install desired Kubernetes version
- add the new VM to the cluster (custom annotations, labels and taints etc are retained automatically)

Agent nodes:

- create new VM and install desired Kubernetes version
- add the new VM to the cluster
- evict any pods that might be scheduled onto this node by Kubernetes before copying custom node properties
- copy the custom annotations, labels and taints of old node to new node.
- cordon the node and drain existing workloads
- delete the VM

### Simple steps to run upgrade

Once you have read all the [requirements](#pre-requirements), run `aks-engine upgrade` with the appropriate arguments:

```bash
./bin/aks-engine upgrade \
  --subscription-id <subscription id> \
  --api-model <generated apimodel.json> \
  --location <resource group location> \
  --resource-group <resource group name> \
  --upgrade-version <desired Kubernetes version> \
  --auth-method client_secret \
  --client-id <service principal id> \
  --client-secret <service principal secret>
```

For example,

```bash
./bin/aks-engine upgrade \
  --subscription-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
  --api-model _output/mycluster/apimodel.json \
  --location westus \
  --resource-group test-upgrade \
  --upgrade-version 1.8.7 \
  --client-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
  --client-secret xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### Steps to run when using Key Vault for secrets

If you use Key Vault for secrets, you must specify a local [kubeconfig file](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) to connect to the cluster because aks-engine is currently unable to read secrets from a Key Vault during an upgrade.

```bash
 ./bin/aks-engine upgrade \
   --subscription-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
   --api-model _output/mycluster/apimodel.json \
   --location westus \
   --resource-group test-upgrade \
   --upgrade-version 1.8.7 \
   --client-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
   --client-secret xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
   --kubeconfig ./path/to/kubeconfig.json
```

## Known Limitations

### Manual reconciliation

The upgrade operation is a long-running, successive set of ARM deployments, and for large clusters, more susceptible to one of those deployments failing. This is based on the design principle of upgrade enumerating, one-at-a-time, through each node in the cluster. A transient Azure resource allocation error could thus interrupt the successful progression of the overall transaction. At present, the upgrade operation is implemented to "fail fast"; and so, if a well formed upgrade operation fails before completing, it can be manually retried by invoking the exact same command line arguments as were sent originally. The upgrade operation will enumerate through the cluster nodes, skipping any nodes that have already been upgraded to the desired Kubernetes version. Those nodes that match the *original* Kubernetes version will then, one-at-a-time, be cordon and drained, and upgraded to the desired version. Put another way, an upgrade command is designed to be idempotent across retry scenarios.

### Cluster-autoscaler + Availability Set

We don't recommend using `aks-engine upgrade` on clusters that have Availability Set (non-VMSS) agent pools `cluster-autoscaler` at this time.

<a name="force-upgrade"></a>
## Forcing an upgrade

The upgrade operation takes an optional `--force` argument:

```shellscript
-f, --force
force upgrading the cluster to desired version. Allows same version upgrades and downgrades.
```

In some situations, you might want to bypass the AKS-Engine validation of your apimodel versions and cluster nodes versions. This is at your own risk and you should assess the potential harm of using this flag.

The `--force` parameter instructs the upgrade process to:

- bypass the usual version validation
- include __all__ your cluster's nodes (masters and agents) in the upgrade process; nodes that are already on the target version will __not__ be skipped.
- allow any Kubernetes versions, including the ones that have not been whitelisted, or deprecated
- accept downgrade operations

> Note: If you pass in a version that AKS-Engine literally cannot install (e.g., a version of Kubernetes that does not exist), you may break your cluster.

For each node, the cluster will follow the same process described in the section above: [Under the hood](#under-the-hood)
