# Upgrade v2.23 to v2.24

## Prerequisites

- [ ] Notify the users (if any) before the upgrade starts;
- [ ] Check if there are any pending changes to the environment;
- [ ] Check the state of the environment, pods, nodes and backup jobs:

    ```bash
    ./compliantkubernetes-apps/bin/ck8s test sc|wc
    ./compliantkubernetes-apps/bin/ck8s ops kubectl sc|wc get pods -A -o custom-columns=NAMESPACE:metadata.namespace,POD:metadata.name,READY-false:status.containerStatuses[*].ready,REASON:status.containerStatuses[*].state.terminated.reason | grep false | grep -v Completed
    ./compliantkubernetes-apps/bin/ck8s ops kubectl sc|wc get nodes
    ./compliantkubernetes-apps/bin/ck8s ops kubectl sc|wc get jobs -A
    velero get backup
    ```

- [ ] Silence the notifications for the alerts. e.g you can use [alertmanager silences](https://prometheus.io/docs/alerting/latest/alertmanager/#silences);

## Steps that can be done before the upgrade - non-disruptive

1. Checkout the new release: `git switch -d v2.24.x-ck8sx`

1. Switch to the correct remote: `git submodule sync`

1. Update the kubespray submodule: `git submodule update --init --recursive`

1. If the cluster config folders are named with different prefixes than `sc` and `wc`, rename the folders to `sc-config` and `wc-config` respectively

1. Run `bin/ck8s-kubespray upgrade both v2.24 prepare` to update your config.

    > [!NOTE]
    > It is possible to update `wc` and `sc` config separately by replacing `both` when running the `upgrade` command, e.g. the following will only update config for the workload cluster:
    > ```bash
    > bin/ck8s-kubespray upgrade wc v2.24 prepare
    > ```

1. Download the required files on the nodes

    ```bash
    ./bin/ck8s-kubespray run-playbook sc upgrade_cluster.yml -b --tags=download
    ./bin/ck8s-kubespray run-playbook wc upgrade_cluster.yml -b --tags=download
    ```

## Upgrade steps

These steps will cause disruptions in the environment.

1. Upgrade the cluster to a new kubernetes version:

    ```bash
    ./bin/ck8s-kubespray run-playbook sc upgrade_cluster.yml -b -e skip_downloads=true
    ./bin/ck8s-kubespray run-playbook wc upgrade_cluster.yml -b -e skip_downloads=true
    ```

1. Ensure control plane nodes only have the `control-plane` taint and not `master`.

    ```bash
    ./migration/v2.24/apply/01-fix-cp-node-taints.sh <sc|wc|both>
    ```

1. **If the cluster runs on Safespring** Update netplan to set default interface as critical

    This is to prevent the interface to release the IP when the DHCP server stop responding for a while.

    Run the following playbook, if the default interface is something else than `ens3` you can add the following flag `-e netplan_critical_dhcp_interface=<interface name>`

    ```bash
    ./bin/ck8s-kubespray run-playbook wc ../../playbooks/set_critical_interface.yml -e netplan_critical_dhcp_interface=ens3
    ./bin/ck8s-kubespray run-playbook sc ../../playbooks/set_critical_interface.yml -e netplan_critical_dhcp_interface=ens3
    ```

    Also add a comment to your config so that any new nodes that will be created get this change as well

    cluster.tfvars:

    ```diff
     "control-plane-0" = {
         "az"          = "nova"
         "flavor"      = "..."
         "floating_ip" = false
         "etcd"        = true
    +    # "netplan_critical_dhcp_interface" = "ens3" # Uncomment this when creating new nodes
     },
    ...
     "worker-0" = {
         "az"          = "nova"
         "flavor"      = "..."
         "floating_ip" = false
    +    # "netplan_critical_dhcp_interface" = "ens3" # Uncomment this when creating new nodes
     },
    ```

## Postrequisite

- [ ] Check the state of the environment, pods and nodes:

    ```bash
    ./compliantkubernetes-apps/bin/ck8s test sc|wc
    ./compliantkubernetes-apps/bin/ck8s ops kubectl sc|wc get pods -A -o custom-columns=NAMESPACE:metadata.namespace,POD:metadata.name,READY-false:status.containerStatuses[*].ready,REASON:status.containerStatuses[*].state.terminated.reason | grep false | grep -v Completed
    ./compliantkubernetes-apps/bin/ck8s ops kubectl sc|wc get nodes
    ```

- [ ] Enable the notifications for the alerts;
- [ ] Notify the users (if any) when the upgrade is complete;

> [!NOTE]
> Additionally it is good to check:
> - if any alerts generated by the upgrade didn't close.
> - if you can login to Grafana, Opensearch or Harbor.
> - if you can see fresh metrics and logs.
