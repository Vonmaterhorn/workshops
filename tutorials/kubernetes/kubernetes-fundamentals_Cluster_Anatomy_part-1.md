# Kubernetes Fundamentals - Cluster Anatomy (Part 1)

The objective of this tutorial is to introduce the core concepts of the Kubernetes cluster. While beginner-friendly,
this document is aimed at those wanting to start knowing Kubernetes a bit more, but have had some contact with it in the
past, even if only as a "black box that runs applications".

The information presented leans heavily on
the [official Kubernetes documentation](https://kubernetes.io/docs/concepts/).
This tutorial attempts to condense a lot of information, and present it in a way that makes it easier on readers to take
on further reading, to learn more about the concepts. As you read on, the links shared along the document will help
better contextualize and explain the concepts being addressed, as these resources often come with diagrams and examples.

By the end of this tutorial, you can expect to know a bit more about:

- The core components of a Kubernetes cluster
- How a Kubernetes cluster works

## Core Components

A Kubernetes cluster is made of one or more nodes. The nodes can either be `Control Plane` or `Workers`, where the first
manages how the second operates. In most production environments, a Control Plane is usually a collection of nodes that
are separated from the Worker nodes, and focuses on making decisions and handling events for the cluster, so that its
state is coherent, and the decisions are carried out effectively.

Below is an overview of this architecture, with the interaction between control-plane and workers.

![components-of-kubernetes](./components-of-kubernetes.svg)

### Control Plane

As the management layer, the Control Plane oversees the state of the cluster. It has the following components:

- **kube-apiserver**: at its core, Kubernetes is a collection of resources managed by an API. The API server is the
  front end for the Kubernetes Control Plane, and each API call either _impacts_ the state of the cluster and its
  objects, or _reflects_ them.

- **etcd**: Consistent and [highly-available key value store](https://etcd.io/) used as Kubernetes' backing store for
  all cluster data. It stores the state of every resource of the cluster, across all nodes of the control plane, as a
  distributed database.

- **kube-scheduler**: Kubernetes wouldn't be of much use if it couldn't decide how to run workloads. The Scheduler looks
  for `Pods` that aren't associated with a node, and attempts to select a valid candidate to run them on.

- **kube-controller-manager**: This is actually a collection of processes that each cater to a specific task. Logically,
  each [controller](https://kubernetes.io/docs/concepts/architecture/controller/) is a separate process, but to reduce
  complexity, they are all compiled into a single binary and run in a single process. Some of these control how the
  cluster handles membership changes, how to handle storage interfaces, authorization inside the cluster, etc. The list
  varies on what the cluster actually needs to "control", but controllers share a design pattern: the *control-loop*,
  which constantly looks for current state, compares it to the desired state, and adjusts as necessary.

- **cloud-controller-manager**: Nowadays, it's often the case that Kubernetes clusters run on a Cloud provider (AWS).
  This controller manages communication with the Cloud resources' APIs, with cloud-specific control logic, and handles
  all abstractions necessary to associate the representation of a resource in Kubernetes, and the respective Cloud
  counterpart. As a
  controller, it has many control-loops that rely on Kubernetes objects' state to adjust Cloud resources, and
  vice-versa. It's often the case that some controllers have the cloud-controller as a dependency, so they can adjust
  the resources they oversee. For example:
  - The *Node controller* is responsible for noticing and responding when nodes go down. If the cluster is running on
    nodes in a Cloud provider, determining if an instance has been deleted in the Cloud, after the node stops
    responding, is necessary in order to roll out the appropriate status changes. The Node controller will rely on the
    Cloud controller to pass along information that helps achieve the desired state.

Typically, you may see the term *Master* being interchangeably used with *Control Plane* to address this type of node.
This is due to how the nomenclature has changed over time, as Master was the initial term used, with Control Plane being
introduced at a latter
time, [to phase out and replace the term Master](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#urgent-upgrade-notes).
Throughout this (and other) tutorials, both terms will always refer to the same type of node, and the same concept of
cluster management/overseeing.

### Nodes

Regardless if it's a Control Plane/Master or Worker node, all Kubernetes nodes run the following:

- **kubelet**: An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod. When
  a Pod is scheduled to run on a node, the kubelet (on the selected node) is given a `PodSpec`, a manifest on what the
  workload is supposed to be, and makes the necessary adjustments to ensure the desired state is met. It'll manage the
  Pod, along with its container(s), for as long as the Pod is scheduled to that node.
- **kube-proxy**: kube-proxy is a network proxy that runs on each node of the cluster, implementing part of the
  Kubernetes
  Service concept. The kube-proxy is a collection of network rules, dictated by what's running on a node, and will allow
  or deny communication inside the cluster, amongst other containers and services, or even with the network outside the
  cluster. These rules are typically IPTABLES rule chains for TCP/UDP traffic, when running on Linux nodes.
- **Container runtime**: The container runtime is the software that is responsible for running containers. Since
  Kubernetes version 1.20 began phasing out the use of Docker runtime, we use
  [containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md) as the container runtime,
  but Kubernetes is compatible with [CRI-O](https://cri-o.io/) and any implementation of the Kubernetes CRI
  (Container Runtime Interface).
  - kubelet is entirely dependent on the container runtime to manage the containers. If there's no runtime, or it's not
    working properly, the node won't be usable to run workloads.

Nodes that run the Control Plane are typically outside the pool of nodes used to run our applications. This is for
separation of duties: Control Plane handles the cluster, Workers handle user-defined workloads. By having applications
running only on Worker nodes, there's reduced exposure of critical core components, and less probability that an
application bringing down the node will affect the overall cluster state (as opposed to having these breaking a Control
Plane node and taking down the API server, or other core components).

# Quick Recap

- Kubernetes clusters are made of Control Plane nodes and Worker nodes.
  - Control Plane is responsible for:
    - Exposing the API server, which handles resource definitions
    - Managing the controllers that observe state and adjust to maintain the cluster the desired state, as well as the
      Worker nodes
    - Maintaining all the state in the ETCD, a distributed key-value store, and keeping it coherent;
    - Running the scheduler, responsible for handling the allocation of nodes to run workloads.
  - Worker nodes are used to run applications that the Control Plane assigns and are often in a node pool separate
    from Control Plane nodes.
  - Both Control Plane and Worker nodes rely on kubelet to manage the container runtimes on which workloads run; and a
    network layer to enable inter-node and API communication.

# What's Next

You should now read
[Kubernetes Fundamentals - Cluster Anatomy (Part 2)](./kubernetes-fundamentals_Cluster_Anatomy_part-2.md).
