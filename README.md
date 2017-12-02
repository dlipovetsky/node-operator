# node operator
Manage system packages and systemd services using Kubernetes.

# Motivation
Installing a container runtime and kubelet is a necessary part of bootstrapping a Kubernetes node. Upgrading the container runtime and kubelet is a necessary part of upgrading the node. Neither is self-upgrading today, though kubelet self-upgrade has been discussed before[0].

The community has dealt with the node upgrade problem in different ways. When an IaaS API is used to deploy a node, it is typically not upgraded, but instead terminated and replaced by a new node[1]. Some operating systems ship with container runtimes and kubelet pre-installed, and upgrade them during a system restart[2].

Replacing instead of upgrading a node is not a practical strategy on bare-metal, and not everyone's operating system of choice ships with container runtimes and kubelet pre-installed. The node operator makes it possible to install and upgrade container runtimes and kubelet using the Kubernetes API.

Even if the node upgrade problem is solved, upgrading nodes in a cluster has other challenges. Upgrading all nodes in parallel can affect cluster workload: in some cases, nodes must be drained before upgrade, and the upgrade itself may disable a node. One well-known strategy is the rolling upgrade: nodes are upgraded in batches, and every batch must pass a post-upgrade health check before the batch is upgraded. Rollback of upgrades is sometimes supported[3]. The node operator orchestrates rolling node upgrades with health checks and rollback.

- [0] [https://github.com/kubernetes/kubernetes/pull/7233](https://github.com/kubernetes/kubernetes/pull/7233)
- [1] [https://kubecloud.io/upgrading-a-ha-kubernetes-kops-cluster-9fb34c441333](https://kubecloud.io/upgrading-a-ha-kubernetes-kops-cluster-9fb34c441333)
- [2] [https://github.com/coreos/container-linux-update-operator](https://github.com/coreos/container-linux-update-operator)
- [3] [https://cloud.google.com/kubernetes-engine/docs/how-to/upgrading-a-container-cluster#rolling_back_a_node_upgrade](https://cloud.google.com/kubernetes-engine/docs/how-to/upgrading-a-container-cluster#rolling_back_a_node_upgrade)

[https://github.com/kubernetes/kubernetes/issues/2884](https://github.com/kubernetes/kubernetes/issues/2884)

# Proposal

The node operator is made up of two components: *nodelet* and *node-controller*.

Nodelet installs and upgrades container runtimes and kubelet on nodes;  it is a daemon that runs on the node. Like kubelet, nodelet is *reconciling*: it continually drives the node's actual state toward some desired state. Nodelet can:
- Install/remove repositories of system packages
- Install/remove system packages
- Place static configuration used by systemd services
- Start/stop systemd services

Node-controller orchestrates rolling upgrades; it runs as a cluster add-on.

Node-controller pushes each node's desired state to the Kubernetes API, and all nodelets pull their desired state from the Kubernetes API.

There are many [configuration management tools](https://en.wikipedia.org/wiki/Comparison_of_open-source_configuration*management_software) (CMTs). Some well-known tools are ansible, chef, puppet, and salt. All support managing system packages and systemd services, but most include much more functionality. The node operator is not meant to replace your CMT. Initially it is just powerful enough to install and maintain container runtimes and kubelet, as well as other Kubernetes dependencies that are available as system packages.

# User Experience
- Use the Kubernetes API to orchestrate rolling upgrades of container runtimes, kubelet, and other Kubernetes dependencies.

# Crazy Ideas
Bootstrap docker and kubelet with a different tool. Deploy the *nodelet* as a DaemonSet. Use *nodelet* to upgrade docker (first ensuring [--live-restore](http://https://docs.docker.com/engine/admin/live-restore/) is enabled) and kubelet. Use *kubelet* to upgrade *nodelet*.

What else is required to upgrade nodes running master or storage components? Do we need to run node-controller in HA, e.g., on all masters, and use leader election?
