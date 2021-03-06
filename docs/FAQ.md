# Questions and answers

The goal of this file is to have a place to easily commit answers to questions
in a way that's easily searchable, and can make its way into official
documentation later.

## Q: When I customize the Ignition generated by the installer, how does the MCO handle that?

This is a supported operation.  Today, the MCO does not have good support for "per node"
configuration, and configuring things like static IP addresses and partition layouts
by customizing the Ignition makes sense.

However, it's important to understand that these custom changes are "invisible" to the
MCO today - they won't show up in `oc get machineconfig`.  And hence it's not
as straightforward to make any "day 2" changes to them.

In the future, it's likely the MCO will gain better support for per-node configuration
as well as tools to more easily manipulate Ignition, so there is less need to edit the
Ignition JSON directly.

## Q: Why are my workers showing older versions of RHCOS?

Today, the MCO only blocks on upgrades of control plane nodes.  `oc get clusterversion` effectively reports the version of the control plane. 

To watch rollout of worker nodes, you should look at `oc describe machineconfigpool/worker` (as well as other custom pools, if any).

## Q: How does this relate to Machine API?

There are two fundamental operators in OpenShift 4 that both include "machine" in their name:

The Machine Config Operator (this repository) manages code and configuration "inside" the OS (and targets specifcally RHCOS).

The [Machine API Operator](https://github.com/openshift/machine-api-operator) manages "machine" objects which represent underlying IaaS virtual ([or physical](https://github.com/openshift-metal3)) machines.

In other words, they operate on fundamentally different levels, but they do interact.  For example, both currently will drain a node.  The MCO will drain when it's making changes, and machine API will drain when a machine object is deleted and has an associated node.

Another linkage between the two is booting an instance; in IaaS scenarios the "user data" field (managed by machineAPI) will contain a "pointer Ignition config" that points to the Machine Config Server.

However, these repositories have distinct teams.  Also, machineAPI is a derivative of a Kubernetes upstream project "cluster API", whereas the MCO is not.

## Q: If I change something manually on the host, will the MCO revert it?

Usually, no.  Today, the MCO does not try to claim "exclusive" ownership over everything on the host system; it's just not feasible to do.

If for example you write a daemonset that writes a custom systemd unit into e.g. `/etc/systemd/system`, or do so manually via `ssh`/`oc debug node` - OS upgrades will preserve that change (via libostree), and the MCO will not revert it.  The MCO/MCD only changes files included in `MachineConfigs`, there is no code to look for "unknown" files.

Another case today is that the SDN operator will extract some binaries from its container image and drop them in `/opt` (which is really `/var/opt`).

Stated more generally, on an OSTree managed system, all content in `/etc` and `/var` is [preserved by default across upgrades](https://ostree.readthedocs.io/en/latest/manual/adapting-existing/).

Further, rpm-ostree supports package layering and overrides - these will also be preserved by the MCO (currently).  Although note that there is no current mechanism to trigger a MCO-coordinated drain/reboot, which is particularly relevant for `rpm-ostree install/override` changes.

If a file that *is* managed by MachineConfig is changed, the MCD will detect this and go degraded.  We go degraded rather than overwrite in order to avoid [reboot loops](https://github.com/openshift/machine-config-operator/pull/245).

In the future, we would like to harden things more so that these things are more controlled, and ideally avoid having any persistent "unmanaged" state.  But it will take significant work to get there; and the status quo means that we can support other operators such as SDN (and e.g. [nmstate](https://github.com/nmstate/kubernetes-nmstate)) that may control parts of the host without the MCO's awareness.
