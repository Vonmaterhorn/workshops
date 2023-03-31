# Kubernetes Fundamentals - Defining Workloads (Part 1)


The objective of this tutorial is to go over how Kubernetes represents workloads. Regardless
of complexity, all workloads are represented by Pods, running code on containers, working
collaboratively in one or more cluster nodes.
Kubernetes offers a wide variety of resources, and an almost infinite number of configurations,
to cater to an application's needs, when it comes to create the perfect infrastructure for it
run on.

By the end of this tutorial, you can expect to know a bit more about:
- The API resource definitions for Pod and Container;
- The more popular and widely used parts of their specification, to create the objects
  that normally take part in an application's infrastructure;
- How to put all of this together to create a Pod manifest.

## The Pod

As the smallest unit of representation of computational work, Pods are the equivalent of a
(logical) host on which applications traditionally run on: it limits what resources the containers
have access to, such as compute, memory, storage, and to a certain degree, network.

Pods aren't limited to running a single container, though. A single Pod can have multiple
containers, like _peas_ in a _pod_, and they'll always be scheduled together, co-located in the
same node, or not at all, should there be no available resources for all of them.

Containers come in different _flavours_, too:
- Regular Containers (also known as Application Containers)
- Init Containers
- Ephemeral Containers

Whether one or more, containers always need to be configured for a `Pod`. Multiple containers can work
together, collaboratively, or they can address different facets of a problem that the application
aims to solve. Some can even run before others, to ensure some settings or environment are configured
before the application kicks off.

### Types of Containers

The `initContainer` is a type of container that runs before the regular containers, useful for
work that needs to be done before the application is ready to start. It can do some groundwork
for the main process(es) to run properly, it can ensure some part of the environment is
configured correctly, for something that couldn't be done in the Docker image; or it can simply
run validations and post information to be captured in the logs, for example. Regardless of what
these initContainers do, they'll **run before the application container(s)**, and if they crash, 
they'll prevent other containers from being launched. Because initContainers run sequentially,
they can prevent regular containers from ever starting if they begin crashing. If a Pod is set
to never restart its containers (when using `restartPolicy: Never`), failing initContainers will
fail the whole Pod.

The `ephemeralContainer` is a relatively new concept, and while not widely used, has
an interesting function: they can be injected in an existing Pod, which make them perfect for troubleshooting
applications. While regular containers normally have some guarantees for availability/execution,
ephemeral containers have none of these, they're truly ephemeral, and never restart if they crash.
EphemeralContainers cannot specify resources to use, but can specify storage and share filesystem
with regular containers, if needed. An important note is that these can't be specified as part of 
a Pod `spec`, they're meant to be launched _ad-hoc_, to debug an application's environment, for example.

The Regular `container` is the main building block for an application. They decouple the application
from the underlying infrastructure, and can request resources and environment, so they have access to
exactly what they need to run. Regular containers are subject to lifecycle rules from Pods, as in, 
they can be set to restart on failure, to ensure applications are fault-tolerant. InitContainers, for
comparison, are able to set resources, but don't have the `lifecycle` concept because they must always
run to completion.

### Lifecycle

Pods start off by being scheduled by the cluster, and allocated resources on a node. When they're initially created, and
containers are still waiting to be able to start (such as, waiting for their image to download, waiting for a node to be
available to run them, or waiting for some other condition), they post a `Pending` status. As soon as containers begin 
to run, the Pod changes to `Running`, and remains in such a state until containers' main process exit. If the exit is 
successful, because containers have run to completion, the Pod will be marked as `Succeeded`, as the Pod has done its 
job and won't be restarted. However, if one or more containers fail (as in, abruptly terminated, or exit with a non-zero
exit code), the Pod will be marked as `Failed`.

Kubernetes has a restartPolicy setting for Pods, to control the lifecyle with a policy applicable to all containers in a
Pod:
- `restartPolicy: Always`: This is the default setting, and state Pod containers should always be restarted when they
  exit, regardless of exit code.
- `restartPolicy: Never`: This setting makes it so containers exiting, for whatever reason, won't trigger a restart from
  the kubelet, and may lead to Pods having a `Failed` status if the containers failed.
- `restartPolicy: OnFailure`: This setting ensures containers are only restarted if they've failed to exit with a 
  successful exit code. This setting is popular for task-oriented workloads, where it's expected for the application to
  exit once it has done its job, or retry if it failed.

Given the Pod's `restartPolicy`, a `Failed` Pod may be restarted, in an effort to comply with desired state. If the 
containers keep exiting in non-successful exit codes, a `CrashLoop` cycle will ensue, for which the cluster introduces a
back-off timer, increasing the hold-off period at each iteration. Pods post a `CrashLoopBackOff` message when in this 
state. This hold-off timer is reset if the container runs successfully for 10 minutes, or longer.

Since Pods aim to represent containers running applications, it's important that they complete whatever they were doing
before shutting down. Containers can be gracefully or abruptly terminated by the Kubelet, but should still be allowed to
clean-up before exiting, whether that's done through the application or not. To this extent, Kubernetes allows for a 
grace period on a Pod's deletion, before it forces its termination, using the setting `terminationGracePeriodSeconds`.
- When a Pod is marked for deletion, the Kubelet will send a signal (typically a TERM, for most runtimes) to the process,
  and wait for the process to exit. If the application has any handlers for these signals, that might mean it'll take 
  steps to ensure some correct clean-up process, which takes time. After the grace period has expired, any remaining 
  containers (read, processes) will receive a forceful termination, using the KILL signal. 
  - There are also ways for which a Pod's containers might run some actions when set to terminate, outside what the 
    application code might do. If you're interested in the concept of running user-defined commands outside an 
    application, for a given container lifecycle event, read more about 
    [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/).
- The grace period is a _per-Pod_ window, and not _per-container_, meaning different types of containers won't result in
  stacking grace periods.
- The default value for `terminationGracePeriodSeconds` is 30 seconds, as of Kubernetes version `1.23`.

### Probes
In addition to process status, the Pod's `Phase` can also be affected by diagnostic results for
running containers. Containers can _misbehave_ and still not exit or crash, so Kubernetes allows for
different mechanisms to probe a container, and evaluate the results against some pre-determined conditions.
These diagnostics are configurable not only for mechanism, but also for periodicity.

The commonly available [probe mechanisms](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#probe-check-methods)
consist of:
- `exec`: Executes a specified command inside the container. The diagnostic is considered successful if the 
  command exits with a status code of `0`. These are particularly interesting when HTTP endpoints can't be used,
  or some other conditions need to be met, like checking for the existence of a file in the filesystem.
- `httpGet`: Performs an HTTP `GET` request against the Pod's IP address on a specified port and path. The 
  diagnostic is considered successful if the response has a status code greater than or equal to 200 and less
  than 400 (`SUCCESSFUL` or `REDIRECTION` responses codes).
- `tcpSocket`: Performs a TCP check against the Pod's IP address on a specified port. The diagnostic is considered 
  successful if the port is open. If the remote system (the container) closes the connection immediately after it opens,
  this counts as healthy.

Probe's `success` or `failure` response help ensuring a Pod's availability or fitness to receive traffic. But these
probes can be helpful at different stages of a Pod's lifecycle, so Kubernetes uses three kinds of probes to cater for
different configuration needs:
- `livenessProbe`: Sets up conditions to ensure the application, running in the container, is running. Note that this
  doesn't mean it's available to receive requests, or ready to work. It merely states that it's _alive_.
  - If this probe fails, the container is restarted (depending on the restartPolicy).
  - If not provided, a Pod's default `livenessProbe` state is `success`.
- `readinessProbe`: Sets up conditions to ensure the application is ready to receive traffic. These probes are specially
  important if the application serves requests, but only if it's _ready_ to do so, as otherwise request could be routed
  but unable to be fulfilled.
  - If the condition fails, the Pod's IP address is not published as a valid routing Endpoint for a given Service (so, 
    it's effectively offline). The state is set to `failure` until the condition is evaluated, to ensure there's no
    window to briefly expose the application before conditions are evaluated. 
  - If not provided, a Pod's default `readinessProbe` state is `success`.
- `startupProbe`: Sets up conditions to validate an application has started. When this is configured, the other probes
  won't start until this succeeds, which is helpful when ensuring readiness and liveness are run on applications ready
  to receive such probing.
  - This kind of probe is relatively new (version 1.20), and while it isn't widely used, it provides a very useful
    validation, specifically for applications that have slow booting sequences, such as legacy applications.
  - Similar to `livenessProbe`, if the condition fails, the container is restarted (depending on the restartPolicy).
  - If not provided, a Pod's default `startupProbe` state is `success`.

## The `Spec`

A lot of what was talked about previously culminates in actually generating a manifest that represents the exact desired
state for a Pod object. The `spec` section of a Pod's manifest can be complex, given the granularity of configurations
that can be set, so taking advantage of defaults can greatly benefit the readability of a manifest.

[Pod resources in the API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#pod-v1-core) are defined
as having:
- An `apiVersion`
- A `kind`
- A `metadata`
- A `spec`
- A `status`

Out of all these, the only we don't set in a manifest is the `status`, as it's the most recently observed status of the 
pod, and is populated by the system, as a Read-only field. For the rest, we've covered in previous tutorials what
`apiVersion`, `kind`, and `metadata` are, so let's focus on the `spec`.

The next few sections can be a bit lengthy, as they intend to provide a brief description over some of the more widely 
used parts of the spec, as well as the fields each object can have. They're collapsible for ease of reading, but 
shouldn't be skipped entirely, unless most of this isn't new to you.

### `containers` and `initContainers`

This is where we configure what, and how, the application container, single or multiple, run. Most importantly, this is
where the environment is set up, and the expectations for lifecycle are laid down. 

<details>
  <summary>Here are the most important fields to set and configure:</summary>

- `args`: If you need to pass arguments to a container's Dockerimage, this is the place. Variables, such as $(VAR) are
  expanded in the container's environment.
  - This is an array of strings.
  - If not provided, the `CMD` of a Dockerimage is used instead.
- `commands`: This is the ENTRYPOINT to be passed to a container, and may use variables (to be expanded in the 
  container's environment). 
  - This is an array of strings.
  - If not provided, the `ENTRYPOINT` of a Dockerimage is used instead.
- `env`: The environment variables to be set for a container. Per the 12Factor 
  [config philosophy](https://12factor.net/config), this is where applications should go for any setting. This field
  configures a list of KEY:VALUE pairs. The values can either be set explicitly, or reference other Kubernetes objects.
  - This is an array of strings.
  - There is no default.
- `envFrom`: Similar to `env` in function, but uses references to other Kubernetes objects to populate the environment
  variables. When a key exists in multiple sources, the value associated with the last source will take precedence. 
  Values defined by an Env with a duplicate key will take precedence
  - This is an array of strings.
  - There is no default.
- `image`: The Dockerimage URL to use in the container. If the registry is not passed, it'll assume the default registry
  of dockerhub.io, or any other fallback that might be configured.  
  - This is a string.
  - There may be a default, but for most cases, this should always be provided.
- Each of `livenessProbe`, `readinessProbe`, and `startupProbe` objects can set:
  - Exactly one of:
    - `exec` has only one field:
      - `command`: Command line to execute inside the container, the working directory for the command is root ('/') 
        in the container's filesystem. The command is simply exec'd, it is not run inside a shell, so traditional 
        shell instructions ('|', etc) won't work. To use a shell, you need to explicitly call out to that shell.
    - `httpGet` has the following fields:
      - `host`: the FQDN of the endpoint to connect to.
      - `httpHeaders`: An array of {`name`: foo, `value`: bar} objects, representing the HTTP headers to pass.
      - `path`: the location of the resource in the host to access.
      - `port`: number between 1-65535, or an [IANA_SVC_NAME](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.txt),
        but no default.
      - `scheme`: protocol to use. Defaults to HTTP.
    - `tcpSocket` has the following fields:
      - `host`: the FQDN of the endpoint to connect to.
      - `port`: number between 1-65535, or an [IANA_SVC_NAME](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.txt),
        but no default.
  - `failureThreshold`: Minimum consecutive failures for the probe to be considered failed after having succeeded. 
    - Integer, defaults to 3. Minimum value is 1.
  - `successThreshold`: Minimum consecutive successes for the probe to be considered successful after having failed. 
    - Integer, defaults to 1. Must be 1 for liveness and startup. Minimum value is 1.
  - `initialDelaySeconds`: Number of seconds after the container has started before liveness probes are initiated.
    - Integer, no default.
  - `periodSeconds`: How often (in seconds) to perform the probe. 
    - Integer, default to 10 seconds. Minimum value is 1.
  - `timeoutSeconds`: Number of seconds after which the probe times out. 
    - Integer, defaults to 1 second. Minimum value is 1.
- `name`: The container needs a name, unique among a Pod's containers, and must abide the DNS rules (only alphanumeric
  characters, up to 65 characters).
  - String, no default.
- `ports`: The container can specify which ports it listens on, to expose that information publicly in the API.
  - This is an array of objects with the following fields:
    - `containerPort`: Number of port to expose on the Pod's IP address. This must be a valid port number.
      - Integer, between 0 and 65536, no default.
    - `hostIP`: What host IP to bind the external port to.
      - string, no default.
    - `hostPort`: For most cases, this is the same as `containerPort`. However, this can be a different if the Pod is
      to be made available outside the cluster, so the host (Worker Node, for instance) must allocate a port for the
      container and map it, so it can route traffic to it. This is a topic we'll address later, with Services.
      - Integer, between 0 and 65536, no default.
    - `name`: The port name, unique within a Pod's containers, and must abide the DNS rules (only alphanumeric
      characters, up to 65 characters).
      - String, no default.
    - `protocol`: Protocol for port. Must be UDP, TCP, or SCTP. 
      - String, defaults to "TCP".
- `resources`: Compute resources to be allocated to the container. This is an important setting, as it defines what the
  containers can consume when operating, and upon exhaustion, will either crash (on OOM), or thrash (on CPU 
  overconsumption). The resource field has two objects, both optional:
  - `limits`: Maximum amount of resources a container can use.
    - No defaults, and if not set, the container runs unbound, using as many resources and the node can allocate.
    - This field has two objects:
      - `cpu`: amount of CPU units to allocate as a limit.
        - String, as either the representation of an Integer or Float, or the amount of millis to use (500m, or 10000m).
      - `memory`: amount of bytes to allocate as a limit.
        - String, using typical [quantity suffixes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory).
  - `requests`: Minimum amount of compute resources required for the container to run. If a node cannot satisfy these
    compute resource requirements, the Kube-Scheduler may be unable to fit a suitable node, which means the container
    will never run.
    - Defaults to `limits`, if omitted.
    - This field has two objects:
      - `cpu`: amount of CPU units to allocate as a limit.
        - String, as either the representation of an Integer or Float, or the amount of millis to use (500m, or 10000m).
      - `memory`: amount of bytes to allocate as a limit.
        - String, using typical [quantity suffixes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory).
- `volumeMounts`: Pod volumes to mount into the container's filesystem. These can range from the node's own storage, 
  external volumes (such as AWS EBS), other Kubernetes objects, and even parts of the API itself.`volumeMounts` have the
  following fields:
  - `mountPath`: Path within the container at which the volume should be mounted. Must not contain ':'.
    - String, no default.
  - `mountPropagation`: Determines how mounts are propagated from the host to container and the other way around. 
    - String, when not set, `MountPropagationNone` is used.
  - `name`: Name of the volume to mount.
    - String, must match the name of a `Volume` configured in the Pod spec.
  - `readOnly`: Mount as _read-only_ if true, read-write otherwise (false or unspecified).
    - Boolean, defaults to false.
  - `subPath`: Path within the volume from which the container's volume should be mounted.
    - String, defaults to "" (volume's root).
    - Read more about it [here](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath).
  - `subPathExpr`: Expanded path within the volume from which the container's volume should be mounted. Behaves 
    similarly to SubPath but environment variable references $(VAR_NAME) are expanded using the container's environment.
    - String, defaults to "" (volume's root). 
    - `SubPathExpr` and `SubPath` are mutually exclusive.

</details>

<details>
  <summary>Some other important or interesting fields in the container spec, but not covered in detail here:</summary>

- `securityContext`: The Pod's containers should run with sufficient, but constrained, security permissions. This is an
  important setting for security best-practices, and better addressed, in its own right, in a different setting.
  - Read more about the field
    [here](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#securitycontext-v1-core).
  - Typically, the field `runAsNonRoot` is set to true, either at the `Container` level, or at the `Pod` level, `spec`.
- `imagePullPolicy`: This is mostly to control when an image should be pull from the registry VS from the node's cache.
  - This is a string.
  - Defaults to `Always` if _:latest_ tag is specified, or `IfNotPresent` otherwise. If set to `Never`, Kubelet will
    launch the container if it has access to the image locally.
- `lifecycle`: As mentioned above, if a container lifecycle hook configuration is to be used, this is the place to set
  it.
  - This is a [lifecycle object](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#lifecycle-v1-core).
  - There is no default.
- `stdin`, `stdinOnce`, and `tty`: These are settings to allocate a session, with a terminal to read from standard input,
  useful when needing to attach to the container for debugging reasons.
- `terminationMessagePath` and `terminationMessagePolicy`: Not really a popular setting, but
  Kubernetes allows for containers to write messages to disk about their final status.
- `volumeDevices`: On some occasions, containers may need access to raw block devices. This describes a mapping of a 
  raw block device within a container, for advanced uses, such as databases.
- `workingDir`: If needed, one can specify, or override the Dockerfile's, container `WORKDIR`. This field allows just 
  that.

</details>

### `nodeSelector`

For the most part, we tend to let the Kube-Scheduler decide where to put a Pod. However, at times, we need to ensure
Pods run on specific nodes, for reasons such as node architecture, rack awareness, or simply to bundle applications
together. Regardless of reason, when passing a `nodeSelector` in the manifest, we're telling the API to honour this
setting, which may result in the Pod not being scheduled at all, if the condition cannot be met.

A `nodeSelector` term consists of **exactly** one label that a node must have, in order for the Pod to schedule. There
are no defaults, or even a set of fields, but there are some 
[suggested labels](https://kubernetes.io/docs/reference/labels-annotations-taints/) that most nodes get, when they join 
the cluster. Just to name a few:
- `kubernetes.io/arch`: node architecture, such as `amd64` or `arm64` , is useful to schedule application designed to
  run with different CPU architectures.
- `kubernetes.io/os`: node operating system, such as `linux`. Very rarely the case for other host OS that's not `linux`,
  but should it be the case for a `windows` application to run on a matchin OS node, this is useful as a selector label.
- `topology.kubernetes.io/region` and/or `topology.kubernetes.io/zone`: the region and availability zone of a node. This
  is actually one of the key labels to help deciding where to put a Pod when there's rack awareness, so applications
  don't all run on the same zone, on multi zone, High Availability clusters.

### `restartPolicy`

We've talked extensively about this, as it's a crucial setting to differentiate how certain Pods. The `restartPolicy` is
a simple string, one of `Always` (the default), `Never`, and `OnFailure`.

### `serviceAccountName`

The Kubernetes Control Plane allows for an Authentication/Authorization layered centered around tokens. Whether you're
connecting to the cluster to run commands, or an application, running in a Pod, needs to access some object, these 
security primitives hold true: only those allowed to access a resources can do so, and everyone else is denied access.

When a namespace is created, the API server automatically creates a serviceAccount object for that namespace, with a
matching name. Pods, and users, that interact with the API using that serviceAccount token, will have access to 
resources inside that namespace. While oversimplified, that is the gist of it. Kubernetes also allows for a finer
configuration of the authorization layer, by using role-based access control (which we'll cover at a later time).

When `serviceAccountName` isn't specified in the manifest, the default is to assume (or mount) the `ServiceAccount` for
the namespace of the object. While the API Server can populate this automatically, it can also be opt-out, by specifying  
`automountServiceAccountToken: false` in the Pod spec (but there are very little reasons to do so).

### `shareProcessNamespace`

This is a very interesting setting, and one that has great potential when
troubleshooting running applications.

Namespaces and cgroups are what enable containers to exist. The possibility of isolating resources at the kernel level,
that process can then use if run on such contexts, has allowed for the container runtimes to take off, and leave 
virtualization behind, in most cases. These namespace can be used by a process, or a user, or network (and even more),
to provide silos for whatever is using it, providing a reliable isolation of resources. Most notable uses are for CPU
and  memory isolation.

Sharing a single process namespace between all containers in a Pod, allows each process to be in the same context, and
see each other. When this is set, containers will be able to view and signal processes from other containers in the same
Pod, and the first process in each container will not be assigned PID 1. This is good for when troubleshooting needs
to access the process for profiling (for example), but comes at a great cost: the assumption that the main container PID
is `0` no longer holds true, and that may impact the ability of containers accurately providing their status to the API
Server.

Optional, and default to false.

You can read more about Linux kernel's namespaces and cgroups 
[here](https://www.nginx.com/blog/what-are-namespaces-cgroups-how-do-they-work/), and its usage inside Kubernetes
[here](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/#understanding-process-namespace-sharing).

### `terminationGracePeriodSeconds`

In the Lifecycle session we covered on this. The `terminationGracePeriodSeconds` allows for Pods to not abruptly
terminate containers, which may be beneficial for application to clean-up before exiting.

Value must be non-negative integer. The value zero indicates delete immediately. If this value is nil, the default grace
period will be used instead (30 seconds).

### `volumes`

While a container may specify that it needs to mount a certain volume to a certain path, the `volume` itself needs to
be specified, when it comes to identifying the object for the Pod to attach.

Here's a list of what volume types Kubernetes [accepts](https://kubernetes.io/docs/concepts/storage/volumes/#volume-types).
- Some interesting volume types are:
  - [ConfigMap](https://kubernetes.io/docs/concepts/storage/volumes/#configmap) and
    [Secret](https://kubernetes.io/docs/concepts/storage/volumes/#secret), types of Kubernetes objects we'll
    cover later;
  - [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir), an ephemeral storage relying on the
    node's own disk.
  - [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath), a mount of the underlying node's own
    filesystem (which should only be used in very specific circumstances, as it can be a security liability).

The most important thing to keep in mind is that the `name` of a `volume` must match the `name` of a `volumeMount`, or
the attachment won't be successful.

## Putting it All Together

We've gone through what a Pod can define. Looking at a Pod manifest, we should now be able to make sense of what's in
there.

Let's take a look at an example Pod, that you can get with `kubectl describe pod`:

<details>
  <summary><i>Pod Description</i></summary>

  ````yaml
  Name:                 example-app-67c778b945-xrlhr
  Namespace:            example
  Priority:             0
  Priority Class Name:  default
  Service Account:      default
  Node:                 i-0c2bb5aaafabf9d43/10.129.80.191
  Start Time:           Thu, 15 Dec 2022 13:14:18 +0000
  Labels:               app=example
                        pod-template-hash=67c111b945
                        type=web
  Annotations:          
                        SHA: fe7cbd4
                        build: example-app-deploy-711
                        cni.projectcalico.org/containerID: 28c4d7f057eb23cb2c5137134b4a1da46145f82f0bba15ba5d02f022ccee9ccf
                        cni.projectcalico.org/podIP: 100.96.83.48/32
                        cni.projectcalico.org/podIPs: 100.96.83.48/32
                        iam.amazonaws.com/role: kube2iam-example-app
                        kubectl.kubernetes.io/default-container: example-app
                        kubectl_plugins-restarted: 2022-10-26 11:27:03 -0400
  Status:               Running
  IP:                   100.96.83.48
  IPs:
    IP:           100.96.83.48
  Controlled By:  ReplicaSet/example-app-67c778b945
  Containers:
    example-app:
      Container ID:  containerd://c1c09dbbe082e4347baa01b5d647ae8bad800fc2195cc68f4a605d77bebfc912
      Image:         registry:8080/example-app:fe7cbd4
      Image ID:      registry:8080/example-app@sha256:45f2e943c363aa394bf6d2e1d44ad30f37aa72735b8499dacfa45042982f6d9e
      Port:          <none>
      Host Port:     <none>
      Command:
        bundle
      Args:
        exec
      State:          Running
        Started:      Thu, 15 Dec 2022 13:14:22 +0000
      Ready:          True
      Restart Count:  0
      Limits:
        cpu:     1
        memory:  512Mi
      Requests:
        cpu:     250m
        memory:  512Mi
      Environment Variables from:
        example-app-cfgmap  ConfigMap  Optional: false
        example-app-secret  Secret     Optional: false
      Environment:         <none>
      Mounts:
        /etc/secrets from etc-secrets (rw)
        /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-qsqt8 (ro)
  Conditions:
    Type              Status
    Initialized       True
    Ready             True
    ContainersReady   True
    PodScheduled      True
  Volumes:
    etc-secrets:
      Type:        Secret (a volume populated by a Secret)
      SecretName:  etc-secrets
      Optional:    false
    kube-api-access-qsqt8:
      Type:                    Projected (a volume that contains injected data from multiple sources)
      TokenExpirationSeconds:  3607
      ConfigMapName:           kube-root-ca.crt
      ConfigMapOptional:       <nil>
      DownwardAPI:             true
  QoS Class:                   Burstable
  Node-Selectors:              <none>
  Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
  Events:                      <none>
  ````
</details>

There are, certainly, some things here that we didn't cover like `affinity`, `tolerations`, or `Priority`. These are
advanced scheduling techniques, which you can know more about 
[here](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).

If we wanted to write a manifest to generate an object such as this one, we could do something like this:

<details>
  <summary><strong>Pod Manifest</strong></summary>

````yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cool-example: "true"
  labels:
    app: example-app
    type: web
  name: example-app
  namespace: example-app
spec:
  containers:
    - args:
        - exec
      command:
        - bundle
      envFrom:
        - configMapRef:
            name: example-app-cnfgmap
        - secretRef:
            name: example-app-secrets
      image: registry:8080/example-app:foo
      imagePullPolicy: IfNotPresent
      name: example-app
      resources:
        limits:
          cpu: "1"
          memory: 512Mi
        requests:
          cpu: 250m
          memory: 512Mi
      securityContext:
        runAsUser: 1000
      volumeMounts:
        - mountPath: /etc/secrets
          name: etc-secrets
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-qsqt8
          readOnly: true
````
</details>

Seems pretty simple now that we know what each of these parts of the manifest mean. Creating a Pod in your local
Kubernetes cluster is as simple as editing some of these settings, like the image (as our registry is now available for
engineers to pull from), and the mounts, both volume and secret/configmaps, and applying with the `kubectl`. 

# Quick Recap
- A `Pod` must define one or more `Containers`.
  - These can be of the `initContainer`, `ephemeralContainer` or the regular `Container` varieties.
    - `InitContainers` and regular `Containers` share the same `spec`, while `ephemeralContainers` have limitations.
  - An `InitContainer` always runs before a regular `Container`, and must run successfully, or it'll block the next
    container from starting.
- A Pod lifecycle is the definition of how it should behave, and handle container exit status, so the desired state is
  honoured.
  - Pods specify a `restartPolicy` to tell the API Server (and Kubelet) what to do when a container exits.
  - Given the policy, a Pod can be restarted or be left as `Failed` or `Suceeded`, which is useful for different types
    of workloads.
  - A Pod can be instructed to be deleted at any time, but there is a configurable grace period, that allows for 
    containers to clean up and exit, rather than be abruptly killed.
    - Containers can have lifecycle hooks to perform actions before they start the main process, or just before they are
      terminated.
    - By default, the API Server gives containers up to 30 seconds to exit, before being killed.
- A Pod can assess its containers state by more than just what the kubelet tells it (if it's `Running` or `Terminated`).
  - For that, we can use probes to figure out if the container is running abnormally (by failing `livenessProbe` 
    conditions), if it's not yet ready to receive traffic (by failing `readinessProbe` conditions), and even if the
    container has finished booting up, and ready to be probed (by using `startupProbes`).
  - Probes can be configured for the amount of retries, the initial wait before probing the container, and timeouts; as
    well as for the mechanism to conduct the probe.
- A Pod's `spec` can define more than just containers:
  - We can use `nodeSelector` to help the Kube-Scheduler allocating our workloads to specific nodes;
  - We can set authorization with `serviceAccountName`.
  - We can specify which volumes our workloads can use for storage.

# What's Next?

Read the [Part 2](./kubernetes-fundamentals_Defining_Workloads_part-2.md) for how Pods can be bundled and managed, to
create different types of workloads. 
