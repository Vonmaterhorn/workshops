# Kubernetes Fundamentals - How Kubernetes _Works_ (Part 1)

The objective of this tutorial is to give a brief introduction on how a Kubernetes cluster works. While the subject is
extensive, and impossible to explain in a short document, the intent is to highlight how Kubernetes enables 
containerized environments, showcase some of the core resources and objects, and give a glimpse of what both the user
and cluster can do. The links are suggested reading to continue on learning about what is introduced here.

By the end of this tutorial, you can expect to know a bit more about:
- The lifecycle of some of the most used objects in a Kubernetes cluster;
- How to interact with a cluster

The information presented leans heavily on the [official Kubernetes documentation](https://kubernetes.io/docs/concepts/).
This tutorial attempts to condense a lot of information, and present it in a way that makes it easier on readers to take
on further reading, to learn more in-depth about the concepts. As you read on, the links scattered along the document will
help better contextualize and explain the concepts being addressed, as these resources often come with diagrams and
examples.

## Observed VS Desired State

After we've seen the core components, their interaction is what makes a Kubernetes cluster be what we need it to be: a
workload orchestrator and manager. From a holistic perspective, everything starts and ends in/with the API-Server. If we
want to run a workload on a container, we start by issuing the definition of what needs to exist, to the API. From then
on, there's a series of interwoven actions that trigger off one another, and culminate with a container executing
whatever commands it was passed. State, and the concept of keeping track of "observed VS desired state" is what makes
everything work. In essence, the cluster can be perceived as an event-driven, stateful orchestrator, that relies on an
API to define resources, the objects it describes, and the ongoing work to ensure their existence and correctness.

There's a series of resources that maybe are somewhat familiar by now: Pod, Deployment, Service, Node. In Kubernetes, 
objects are _persistent entities in the ecosystem, and form the building blocks to represent the state of the cluster_:

>A Kubernetes object is a "record of intent"--once you create the object, the Kubernetes system will constantly work to
ensure that object exists. By creating an object, you're effectively telling the Kubernetes system what you want your
cluster's workload to look like; this is your cluster's desired state.

_in [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)_

Working off of this concept, we can easily bridge into the **object manifest** concept.

### Object Metadata and Spec

Say you wish to run a simple task in a containerized environment, for e.g., a simple `echo "hello world!"`. This is your
_will_. To convey this to a Kubernetes cluster, however, you need a _way_. In enters the `Pod` resource, and the ability
to create Pod objects.

All Kubernetes objects have a `Metadata` and a `Spec` fields:
- The `Metadata` field has the _name_ of the object, along with the _namespace_, if applicable, that helps create
  non-ambiguous, object-bound names, used to identify objects inside the cluster. In the Metadata field there's also
  _labels_ and _annotations_, that act as object attributes useful to identify them in groups.
- The `Spec` field is responsible for determining the characteristics of the object, the _desired state_. This
  definition is tightly bound to the type (or `Kind`) of object in question, which is defined in the API.

When creating an object, these must be included, and follow the specification of the resource in the API. Incomplete or
erroneous `spec` will lead to API rejecting the request.
To specify which resource the manifest has a `spec` for, you provide the `kind`  and the `apiVersion` fields, which
tells Kubernetes of the kind of object to create, and which version of the API it should be created with, respectively.
Manifests are typically written using YAML (files with `.yaml` or `.yml` extension), though JSON is also accepted.

### Object status

>The status describes the current state of the object, supplied and updated by the Kubernetes system and its components.
The Kubernetes control plane continually and actively manages every object's actual state to match the desired state you
supplied.

_in [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)_

When a manifest is applied to a cluster, the object it described and created will go through many phases until it
eventually succeeds or fails. If the current status doesn't match the desired status, Kubernetes will attempt to
reconcile the two, either indefinitely, or until it reaches a threshold and backs-off. This is what makes Kubernetes
_work_, this loop of attempting to always meet the desired state.

## Workload Definitions

A container is the base unit of a workload, as containers are where actual code runs on. A `Pod`, in turn, is the 
smallest computing unit that can be represented and managed in a Kubernetes cluster. Pods can be deployed to a cluster, 
and will rely on one or more containers to carryout whatever task, according to the specification passed. Pods are 
scheduled to nodes, and all containers that a `Pod`defines run on the same node, and are scheduled at the same time, 
in a shared context (we'll go more in-depth on this subject at a later time). Pods are isolated from each other through 
namespaces, but can also be further compartmentalized if necessary. A container running in a Pod cannot interact with 
another container in a different Pod without going through the network, even if both Pods are scheduled to the same host
. Containers inside the same `Pod`, however, may be able to interact directly in different ways, but for the sake of 
simplicity, assume this is configurable, and not something you get by default. It'll be covered when container storage 
and context are addressed.

Here's how a simple `Pod`, that will run the `echo "hello world!"` we want, can be defined:
````yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
spec:
  containers:
  - name: hello-world
    image: alpine
    command:
    - echo
    - "hello world!"

````

This simple YAML, when applied to a cluster, will create a `Pod`, named _hello-world_ in the _default_ namespace (which
is where objects typically go when no _namespace_ field is specified). That Pod will have a container named
_hello-world_, which will run a specific command we give, `echo "hello world!"`, on an `alpine` OS environment (minimal
[Linux distro Docker image](https://hub.docker.com/_/alpine)). When the manifest is passed to the cluster, it'll create
the object, the container will execute the command, which will be outputted to STDOUT, and captured as a _log_ of the
Pod:

````shell
❯ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
spec:
  containers:
  - name: hello-world
    image: alpine
    command:
    - echo
    - "hello world!"

EOF
pod/hello-world created

❯ kubectl get pods hello-world -w
NAME          READY   STATUS              RESTARTS   AGE
hello-world   0/1     ContainerCreating   0          3s
hello-world   1/1     Running             0          3s
hello-world   0/1     Completed           0          4s

❯ kubectl logs hello-world
hello world!
````

If you attempt to create a Pod this way in a Kubernetes cluster, you'll see that after the Pod is launched, the 
container [completes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-states) almost 
immediately (it's a simple `echo` command that takes no time to execute). This makes the container enter the 
`Terminated` state, which in turn makes the Pod enter a `Failed` state. Since the Pod's no longer in the desired state, 
as defined by the manifest, it'll be restarted immediately. 

This happens because Pods have a policy on when to restart, based on the container exit status, which dictates what
Kubernetes should do on a given a container exit status. 
By [default](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy), a Pod has a 
`restartPolicy: Always`, which means if the container isn't running, it should be restarted, regardless of how it 
exited. At every restart, the container immediately exits, which promptly causes the container to be restarted again.
This causes a loop, a `CrashLoopBackOff` is posted as the Pod status, and the Kubelet begins increasing retry period,
incrementally, to save resources.

````shell
❯ kubectl get pod hello-world
NAME          READY   STATUS             RESTARTS      AGE
hello-world   0/1     CrashLoopBackOff   6 (91s ago)   7m7s
````

If we change the _restartPolicy_ to `OnFailure`, we'll see that the container won't be restarted, because the container 
exit successfully. The Pod will be in `Succeeded` state, showing a `Completed` container state, which matches our 
desired state:

````shell
❯ kubectl apply -f - --context staging <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
spec:
  restartPolicy: OnFailure
  containers:
  - name: hello-world
    image: alpine
    command:
    - echo
    - "hello world!"
EOF
pod/hello-world created

❯ kubectl get pod hello-world
NAME          READY   STATUS      RESTARTS   AGE
hello-world   0/1     Completed   0          18s

❯ kubectl get pod hello-world
NAME          READY   STATUS      RESTARTS   AGE
hello-world   0/1     Completed   0          56s
````

This is just a very simple example to showcase how Kubernetes' "status" model heavily dictates all actions that go on in
a cluster. There's a lot more on [Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle) and
overall configuration of a Pod and container, and we'll cover that in other tutorials.

### Types and Hierarchy of Workload Resources

If containers are mapped to a Pod, what are Pods mapped to? It's very unusual to launch stand-alone Pods, so they are
normally associated with other objects that can manage them:
- For one-of executions of some arbitrary command, Pod can be launched via
  [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/). These are often created by timed
  triggers or schedules ([CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)), or _ad hoc_,
  by something or someone. A Job, upon successfully completing its task, won't be retried. However, Jobs can be
  configured to retry until they either succeed, or fail after a certain number of retries, resulting in the Job failing
  entirely.
- For prolonged executions, such as an application or daemon, Pods typically need to run until they're told to stop, or
  fail. This is normally the case for [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/),
  which in turn are normally managed by [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).
  - Deployments and ReplicaSets are not interchangeable terms, but convey a similar concept: they control how and when
    persistent workloads are updated, and define the desired state for much of the definitions ReplicaSets and Pods can
    have, respectively. One of the most known features of these Deployments is the
    ability to scale an application (in or out), and to stop, start, and restart that application.

While there's a lot more to cover on this subject, the important notion to keep is that a container doesn't exist on its
own in the Kubernetes system. It's always associated with a Pod, which itself is a manageable object in Kubernetes.
To control Pods, we typically use Deployments, to specify how they should run, how many to run, and be able to update
some of their desired states, so the cluster adjusts accordingly; and Jobs, to run one-of workloads, either _ad-hoc_ or
on a schedule, that are only retried if they fail, and only for a pre-determined amount of times.

# Quick Recap
- Every object in Kubernetes has a desired state, and an observed state (or status). Kubernetes will work to ensure
  these match.
- Objects' `spec` come from the resource API definition. When creating a manifest to create, say, a `Pod`, the
  definition of what a Pod should look like, is in the API.
  - If the provided manifest doesn't follow the specification, the API-Server will reject the request to apply or
    update it.
  - There are defaults for some of the `spec`, and manifests should typically specify only overrides to the defaults.
- A Pod defines the smallest piece of computing abstraction in the Kubernetes system. A Pod, in turn, specifies its
  container(s) lifecycle rules, what they run, their name.
  - A Pod, as an object in the system, is rarely managed on its own, and it's normally associated with:
    - `Deployments`, if the workloads are expected to run indefinitely, and be fault tolerant;
    - `Jobs`, if the workloads should run _ad hoc_ or on a schedule, with the possibility of being fault tolerant.

# What's Next

You should now read
[Kubernetes Fundamentals - How a Cluster _Works_ (Part 2)](./kubernetes-fundamentals_How_a_Cluster_Works_part-2.md).
