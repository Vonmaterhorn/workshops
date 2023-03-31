# Kubernetes Fundamentals - Cluster Anatomy (Part 2)

In the previous tutorial, we've looked at the core components of a Kubernetes cluster. While those are the only real
fundamental pieces to have a functional cluster, there's a lot more components that each cluster should have, in order 
to actually work as we want and expect. What we'll see next are additional components that most clusters often have
set up, as they provide integral support to some of our platform service requirements.

## Other Components

Outside the Control Plane, there's a plethora of services and concepts that enhance a Kubernetes cluster. When starting
up a new cluster from scratch, most Kubernetes manager solutions will start some amount of these:

### Networking and Network Policy

Kubernetes clusters have their own network. Internally, a cluster will use private addresses (class A, B, or C, 
depending on addressable space needed), so containers can interact among themselves and with the API-server. 
Networking in Kubernetes is often handled by Network Overlays (such as 
[Calico](https://projectcalico.docs.tigera.io/about/about-calico), which is an industry standard), which implement
the Container Network Interface ([CNI](https://github.com/containernetworking/cni)). These are responsible for handling 
the typical Network-related responsibilities: network interface allocation and management, network policies. Some can be
incredible powerful and even offer [eBPF](https://ebpf.io/) capabilities, to manage profiling, tracing, and monitoring
of application from the network layer directly.

### Service Discovery

Kubernetes relies heavily on DNS for the addressable space. While Pods have IP addresses, the concept of Services 
(which we'll dive in detail at a later time) also has DNS names to facilitate service discovery. This often comes from 
the [CoreDNS](https://coredns.io/) service, installed along with the cluster, that provides internal DNS service, to be 
used by Pods that want to contact each other relying on dynamically updated name-to-IP resolution.

### Observability

While not an Addon that normally comes as part of a newly built cluster, observability tools are a must-have in a 
Kubernetes cluster. Solutions are plenty, and can cater to specific needs, but at their core, most observability stack
solutions will give you:
- A way of getting cluster information
- A way of getting runtime metrics
- A way of interacting with gathered data (metrics)
- A way of visualizing that data
- A way of creating alerting actions based on metrics
- A way of capturing container output

While there are some solutions that offer a full stack, most observability stacks are formed by a collection of 
solutions that provide features such as:
- Metric exporters ([node-exporter](https://github.com/prometheus/node_exporter), 
  [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics))
- Interface to query your cluster's, and applications', scraped metrics 
  ([Prometheus](https://prometheus.io/))
- Dashboards ([Grafana](https://grafana.com/))
- Log shipping and parsing, with an interface to interact with logs 
  ([ELK stack, along with Beats](https://www.elastic.co/what-is/elk-stack))
- An interface to create alerts and notify your preferred communication channels 
  ([AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/))

These are just a few examples of some parts that might make the whole of a typical observability stack. At large, most
use some of these, as well as others, like NewRelic. Here's how the stack might look like for production clusters:
- **Metric gathering**:
  - Application metrics are sent to NewRelic, via APM. Applications use a gem, 
    [newrelic_rpm](https://github.com/newrelic/newrelic-ruby-agent) to push application metric data to NewRelic.
  - System metrics are gathered in two ways:
    - NewRelic has an agent running in the cluster (`newrelic-infra`), on all nodes, capturing system information, querying the API server
      directly, and shipping the data to NewRelic. 
    - Prometheus, similarly, captures system data through API. For other infrastructure systems, such as Elasticsearch,
      kafka, the metrics are captured by scraping metric endpoints, populated by dedicated exporters.
      - You can read more about the [exporter concept](https://prometheus.io/docs/introduction/glossary/#exporter) for 
        Prometheus, by the general concept holds for other systems, too.
- **Log Gathering**
  - Logs (now) live exclusively in NewRelic. Similar to metrics, there's an agent _tailing_ all application containers
    for their log outputs, and a log processor ([fluent-bit](https://docs.fluentbit.io/manual/installation/kubernetes)) 
    then sends them to NewRelic log facility.
    - Previously, we relied on another log processor (`fluentd`) for this, when we used Papertrail.
    - The NewRelic agent is capable of adding metadata to each log message, adding context from where it comes, making
      it easier to search through the logs.
- **Data Representation**
  - As far as interfaces to interact with data, we have, again two ways for that:
    - Grafana comes as a part of our Kube-Prometheus stack, and is the standard interface to interact with 
      Kubernetes-related system metrics. We also have Elasticsearch and Kafka metrics here.
    - NewRelic can be used for much of our kubernetes footprint, from system, to application metrics. However,
      Elasticsearch, and Kafka metrics are exclusively in Grafana.
      - NewRelic does provide the benefit of being able to present both logs and metrics in a single interface.
    
### Cloud Services

For clusters that are running on Cloud resources, some services are absolutely essential. Controllers that manage 
storage, RBAC, load balancing, and many other services; are fundamental in managing the automated operations a cluster
goes through. Here are some of the most popular, just to name a few:
- [EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) to manage the automated provisioning and
  management of EBS volumes in AWS, which are then used as storage backends for Pods, and help persist data even on
  node failure.
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/) to manage the
  automated provisioning of Application (and Classic) load balancers in AWS, so applications can be exposed outside
  the Kubernetes clusters, either to the Internet, or inside our VPC, accessible via VPN.
  
### Cluster Services

We call cluster services the collection of applications that enhance the cluster capabilities, or improve on existing
Kubernetes capacities. One of the most notorious examples is the 
[cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler), in which we rely on to
scale nodes in and out of the cluster, to cope with the ebb and flow of compute resource necessities (which we'll see 
in-depth how they can vary from the applications' workloads).
Because our Kubernetes clusters are running in the Cloud, the autoscaler observes the cluster state to determine if new
nodes are needed, or if there are underutilised nodes that can be safely removed from the cluster, and then handles 
communication with AWS APIs to request new instances to join the cluster as nodes, as well as issue termination of 
unneeded instances, to keep the cluster's capacity planning as expected.

Other examples of Cluster Services are Security agents and Policy agents. These are crucial for keeping tabs on
compliance and early detection of attacks and vulnerabilities; and enforcing rules to keep the applications and the
whole environment safe. Some examples of these are:
- [Lacework](https://www.lacework.com/): Security and compliance agent, for CIEM/CWPP/CSPM.
- [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/): OpenPolicy Agent (OPA) solution to enforce
  rules on Kubernetes cluster's resources, according to our organization's policies on security and best practices.

# Quick Recap
- Clusters typically have a lot of addons, services that enhance or ensure the set of capabilities of the base 
  Kubernetes functionalities.
  - For cluster running on, or interacting with, Cloud resources, there's a set of components that make the Kubernetes
    cluster be able to automatically provision and destroy Cloud resources, based on the "control-loop".
    - The cluster-autoscaler
    - The AWS controllers for EBS storage, IAM access, ALB/ELB, etc.
  - One of the key components of a Kubernetes cluster is its instrumentation.
    - A cluster often has a set of tools for observability, that provides insights on how the cluster, and the workloads
      running there, are faring.
      
# What's Next

You should now read
[Kubernetes Fundamentals - How a Cluster _Works_ (Part 1)](./kubernetes-fundamentals_How_a_Cluster_Works_part-1.md).
