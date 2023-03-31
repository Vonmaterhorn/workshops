# Kubernetes Fundamentals - Defining Workloads (Part 2)

Continuing the previous installment of _Defining Workloads_, the goal of this tutorial is to show how Pods can be 
managed to create highly available, and scalable, applications, or to offer guaranties to repeating tasks, running on a
schedule. We've seen what Pods are capable of doing, now we'll focus on how to use them as the resources to take care of
some more complex use cases, and see the real potential of Kubernetes as a workload management and scheduling platform.

# The Sets

As we've seen, Pods are very good at maintaining a group of containers working as intended. The Pod resource defines the
object lifecycle, and to this extent, it'll take measures to ensure the desired state is accomplished, meaning it is 
capable of maintaining an application running, in a very literal sense. As containers do not represent a Kubernetes 
entity in this ecosystem, the Pod is only capable of managing their "life expectancy", through the container lifecycle,
but lack any meaningful way of ensuring applications stay available through container failure, which is what we've come 
to expect of highly available, fault-tolerant systems.

A Pod doesn't self-heal in a broader sense of the word, as they aren't capable of controlling their own persistence:
- A Pod object won't survive a Node failure: if the underlying infrastructure fails, the Pod object is deleted, and the 
  API Server won't take any steps to recreate it, and reschedule it somewhere else.
- A Pod that exhausted its resources, and is killed as a result, won't come back to continue its job. Once it's been
  evicted from a Node, it's gone.

Now, this isn't to say self-heal isn't something achievable with Pods, it most certainly is. But in order to create such
capabilities, we need to abstract another layer, and introduce controllers that maintain a state for a number of Pods
that share a common goal. This is where we introduce ReplicaSets, StatefulSets, and DaemonSets.

## StatefulSet

The first controller we're covering is the `StatefulSet`. In Kubernetes, state is one of the core principles that make
the system aware of changes, and being able to adjust accordingly. It also provides persistence through failure of the
system's components. Some applications benefit of having the same _stateful_ characteristics in their infrastructure,
ensuring the Pods that make up for the application always run in a familiar setting.

StatefulSets maintain Pod identity settings that persist through rescheduling. This means that a StatefulSet's Pod can
be rescheduled to the same node, if it exists, and will have access to its persistent storage. But most importantly,
StatefulSets provide order to a Pod's creation and deletion, ensuring the scaling or rescheduling happens in a fixed,
serialized way for creation (and inverse, in deletion), from Pods `0` through `N-1`, where `N` is the size of the set.
- _Why is this useful?_: When applications are supposed to have a consistent state to their infrastructure, ensuring the
  order by which the application's Pods are modified is important. In a StatefulSet, the Pod `<name>-0` is the first to
  be created, which means it's the first to have its persistent storage provisioned for it, the first to receive network
  presence 
  (through a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) and
  FQDN that includes the service domain), and receives ordered labels that might help distinguishing it among others in
  the set. When being deleted, the Pod `0` is the last to be removed.

StatefulSets bring order to the Pod lifecycle management, ensuring Pods are only created/modified/deletes in a 
serialized fashion. A Pod is only modified when its predecessor has completed the change is operational again.

### Uses for StatefulSets

By virtue of having persistent storage in its own concept, StatefulSet are typically used for applications whose storage
contains important data that should be kept across Pod reschedules. Database implementations in Kubernetes often use
StatefulSets, but any application that needs to store some kind of state, or stateful datasets, outside of a database,
greatly benefits from StatefulSets.

## DaemonSet

The `DaemonSet` controller is pretty straightforward: _what if we wanted to ensure a certain Pod has copies across all 
nodes in the cluster_?

A DaemonSet ensures each node runs a copy of a Pod, save some exceptions that can be configured to the scheduling rules.
Whenever a new node joins the cluster, the DaemonSet automatically schedules a Pod to that node. DaemonSets behave very
much like Deployments (which we'll cover later in this tutorial), controlling Pod lifecycles for applications that
should persist through failures, and are typically stateless (or require no persistent storage for the Pods).

### Uses for DaemonSets

There are some types of applications that take advantage of being present in every node of a cluster, of in every group
of nodes. Most examples are tied to Kubernetes-specific use-cases, such as needing controller applications running on 
each node to ensure other Pods can take advantage of it; or agent-like applications that need to run something in every
instance of the fleet (such as observability tools).

## ReplicaSet

The `ReplicaSet` is the most popular type of Pod controller. As the name suggests, it provides us with a way of creating
and maintaining replicas of a Pod, with many configurable settings on how to do so.

A ReplicaSet defines a `template` of the Pod object it wants to manage. This template is simply a `Pod.spec` that serves
as the blueprint to spawn Pods associated with the set. Along with the Pod definition, ReplicaSets also define the
number of replicas to run of a given Pod (by default, this number is `1`). They're capable of determining what to do 
when the desired Pod count doesn't match the current count: they'll trigger the creation of new Pods, or the deletion of
existing ones, depending on whether the observed state is under, or over, the desired state, respectively. ReplicaSets 
are also able to roll out Pod `spec` changes, in a sane, non-disruptive way, but this is limited by some immutable 
fields, as we'll see in a moment, that will halt any attempt of changing the ReplicaSet's Pod `spec`.

Pods owned by a ReplicaSet are marked with a reference to the `ReplicaSet` object that manages them. This allows them to
keep track of their Pod's states, and act on state deviations. In order to figure out which Pods belong to them, 
ReplicaSets specify a `selector` field, with a condition to figure out which Pods to acquire and reference, based on
labels. Pods matching these labels will be eligible to be acquired, granted they're not already belonging to another
ReplicaSet.
 - *Edge Cases*: Since Pods aren't typically standalone in the Kubernetes system, it's very rare that a ReplicaSet can
  acquire Pods hanging around in a cluster. But if there are any unreferenced Pods that have labels matching the 
  `selector` of a ReplicaSet, those can be acquired, and referenced, even though they weren't spawned by the 
  ReplicaSet's controller. This may lead to a ReplicaSet having Pods with different naming patterns, for instance. This
  is a very farfetched situation, but one that showcases how Pods are managed, and the edges cases to some of 
  Kubernetes' functionalities that one can use in very specific scenarios.
    - You can read more about this in 
    [Non-Template Pod acquisitions](https://kubernetes.io/docs/concepts/workloads/controllers/ReplicaSet/#non-template-Pod-acquisitions).

### Scaling a ReplicaSet

One of the interesting use-cases for a ReplicaSet is to represent an application that can scale in and out. Since 
ReplicaSets state their desired Pod count, changing the number of replicas will create, or delete, Pods accordingly. 
Patching the `replicas` field in the `spec` will trigger the corresponding actions within the cluster:
- Adding a new Pod is done by creating a new `Pod` object, that follows the `spec.template` of the `ReplicaSet`, and
  adding an `ownerReferences` field to the `Pod`'s `metadata`.
- Removing a Pod is done by selecting a `Pod` object, owned by the ReplicaSet, and removing the labels used by the 
  `selector`, or simply deleting the object entirely. While the former is somewhat uncommon to occur, the latter is what
  happens when a ReplicaSet _scales in_. To decide which Pods to delete, the controller sorts them based on the 
  following criteria:
  1. Pods in _Pending_ state take precedence over those in _Running_ state;
  2. Pods in a ReplicaSet that are co-located in the same Node are preferred to those running alone;
  3. Newer Pods (read, created more recently) are selected first for deletion.
  
  If no Pods fit the above criteria, a random Pod is selected for deletion.

The ReplicaSet controller automatically does this for us, whenever we scale in or out. But, for advanced used, we can
manually add or remove a Pod from the set by changing its labels. There are, however, immutable fields in a ReplicaSet,
as well as in Pods, that will stop any attempt of rolling out a change.

### Immutable Field and Non-Propagating Changes 

A `ReplicaSet` is a very simple resource: in addition to the three fields every object must have, it defines a `spec`
consisting of:
- `replicas`: The number of Pods to manage.
- `selector`: A [LabelSelector](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/label-selector/#LabelSelector)
  to specify how to look for Pods that should be managed by the ReplicaSet.
- `template`: The "Pod manifest" (over-simplification), which will be used to spawn new Pods when scaling out.
- `minReadySeconds`: The number of seconds for Pods to be considered _Ready_, after being created and running correctly.

Consider this simple `ReplicaSet` manifest, that creates a REDIS cluster with two nodes:
````yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
````

Some fields in a ReplicaSet cannot be changed for an existing object. A running `ReplicaSet` object cannot have its
`selector` field modified, as it defines which Pods it'll acquire and manage. If successful, this change could break the
condition, but since the Pod still had the references to the ReplicaSet, they'd still be related, which would result in
an erroneous state in the Kubernetes system. If you try to change the `spec.selector.matchLabels.tier` to `redis` 
you'll get:
````shell
The ReplicaSet "frontend" is invalid:
* spec.template.metadata.labels: Invalid value: map[string]string{"tier":"frontend"}: `selector` does not match template `labels`
* spec.selector: Invalid value: v1.LabelSelector{MatchLabels:map[string]string{"tier":"redis"}, MatchExpressions:[]v1.LabelSelectorRequirement(nil)}: field is immutable
````
- This is a simple validation that the API Server provides, by ensuring a `ReplicaSet` selector labels will match the 
labels on the Pods it manages.

But if you try to change both `spec.selector.matchLabels` and `spec.template.metadata.labels`, so they match, the API
will also reject the change:
````shell
The ReplicaSet "frontend" is invalid: spec.selector: Invalid value: v1.LabelSelector{MatchLabels:map[string]string{"tier":"redis"}, MatchExpressions:[]v1.LabelSelectorRequirement(nil)}: field is immutable
````
- As stated above, this would result in a ReplicaSet having a selector condition that wouldn't match the labels in the
Pods it owns.

While this change to labels is somewhat uncommon to occur, there are other modifications to a ReplicaSet's Pod that 
makes sense to do fairly regularly, such as container image tags. If a ReplicaSet is supposed to represent an 
application, then we should be able to update the application at any given time, and have the ReplicaSet _propagate_
that change.

Consider the manifest above. Let's say we want to bump the REDIS version to a more recent one, using the tag `v5`. That
resource modification can be applied, and the API won't bat an eye:
````shell
vagrant@worker1:~$ kubectl apply -f ReplicaSet.yaml
ReplicaSet.apps/frontend configured
vagrant@worker1:~$ kubectl describe ReplicaSet frontend
Name:         frontend
Namespace:    default
Selector:     tier=frontend
Labels:       app=guestbook
              tier=frontend
Annotations:  <none>
Replicas:     2 current / 2 desired
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  tier=frontend
  Containers:
   php-redis:
    Image:        gcr.io/google_samples/gb-frontend:v5
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
````

This looks great, let's see how the Pods are faring:

<details>

````shell
vagrant@worker1:~$ kubectl get Pods
NAME             READY   STATUS    RESTARTS   AGE
frontend-pvz4f   1/1     Running   0          42m
frontend-82d8n   1/1     Running   0          56m
vagrant@worker1:~$ kubectl describe Pods frontend-pvz4f
Name:             frontend-pvz4f
Namespace:        default
Priority:         0
Service Account:  default
Node:             master/192.168.56.10
Start Time:       Fri, 23 Dec 2022 11:17:11 +0000
Labels:           tier=frontend
Annotations:      cni.projectcalico.org/containerID: 1c59690a0f9d74373f9f24f7f7fda5624aba5f1be39b72b02b2f613c667aae99
                  cni.projectcalico.org/podIP: 10.1.219.81/32
                  cni.projectcalico.org/podIPs: 10.1.219.81/32
Status:           Running
IP:               10.1.219.81
IPs:
  IP:           10.1.219.81
Controlled By:  ReplicaSet/frontend
Containers:
  php-redis:
    Container ID:   containerd://d83e90af947c00e21b87d80e01f93f336f0ae03b22e3ce461dd7ff04ecb4c40b
    Image:          gcr.io/google_samples/gb-frontend:v3
    Image ID:       gcr.io/google_samples/gb-frontend@sha256:50b22839aaf6a18586d6751e8963cf684c27b9873ca926df22cdf88ed4452615
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 23 Dec 2022 11:17:31 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-6fv4p (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-6fv4p:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
````
</details>

Well, that's not what we wanted. It seems that the changes we promoted to the Pod's `spec` didn't roll out, even though
the API Server acknowledged the modification we did to the `spec`. So why wasn't this change propagated, leading to the
Pod's replacement with a Pod that fits this `spec`? Remember that a `ReplicaSet` controller watches over its `Pod` 
objects, and from that perspective, their state is fine: they're still running, as there's no change to their liveness 
probes or container status, so there's nothing to do. It won't propagate the changes because that's not its role, it 
won't care for their Pod's `spec` changes. If you were to scale out this ReplicaSet, new Pods will be launched using the 
updated template:
````shell
vagrant@worker1:~$ kubectl scale ReplicaSet frontend --replicas 3
ReplicaSet.apps/frontend scaled
vagrant@worker1:~$ kubectl get pods -o jsonpath='{range .items[*]}{@.metadata.name}:{"\t"}{@.spec.containers[*].image}{"\n"}{end}' --sort-by=.metadata.creationTimestamp
frontend-82d8n:	gcr.io/google_samples/gb-frontend:v3
frontend-pvz4f:	gcr.io/google_samples/gb-frontend:v3
frontend-6hxv2:	gcr.io/google_samples/gb-frontend:v5
````

So, all things considered, ReplicaSets are useful to manage Pods, but they're somewhat ineffective at managing 
application aspects that one would need on a regular basis. So, is there an alternative to ReplicaSet, that lets us
update a collection of Pods, and rolls out these changes for us, while maintaining the application available and 
operational throughout the update?

# The Deployment

Just like ReplicaSets provide a way of managing Pods, Deployments provide a way of managing ReplicaSets, in a way that
updates can be rolled out as we intend, in a configurable way. Deployments are the _de facto_ resource to manage
stateless applications, as they're capable of managing ReplicaSets to provide scalability and correctness to the system.

The `deployment` object has the following fields:
- `apiVersion`, `kind`, and `metadata` (mandatory fields);
- `spec`
  - `minReadySeconds`: Number of seconds to wait for a new Pod, before it's considered available and ready.
    - Defaults to `0`.
  - `paused`: Rollouts can be stopped, so this field indicates that the Deployment is currently paused.
    - Will only appear if the Deployment is paused. Shouldn't be passed in a manifest.
  - `progressDeadlineSeconds`: The maximum time in seconds for a deployment to make progress before it is considered to
    be failed.
    - Defaults to `600` seconds.
  - `replicas`: Number of Pods the Deployment should have.
    - Defaults to `1`.
  - `revisionHistoryLimit`: Number of old ReplicaSets to keep around, to allow `rollback`.
    - Defaults to `10`.
  - `selector`: The same as the ReplicaSet's `selector`. Must match the labels in the Pod template.
  - `strategy`: Strategy to use when replacing Pods.
    - `type`: One of `Recreate` or `RollingUpdate`. When `Recreate` is used, new Pods are only launched after **all**
      old ones have been terminated. `RollingUpdate` means new Pods replace the old ones, at a configurable, controlled
      pace.
      - Default is `RollingUpdate`.
    - `rollingUpdate`: When the `type` is `RollingUpdate`, this configures the strategy parameters.
      - `maxSurge`: The maximum number of Pods that can be scheduled above the desired number of Pods.
        - Can be a number or a percentage. Defaults to `25%`.
      - `maxUnavailable`: The maximum number of Pods that can be unavailable during the update.
        - Can be a number or a percentage. Defaults to `25%`.
  - `template`: Manifest for the Pods this Deployment creates.

When we create a `deployment` object, we're defining the desired state for a collection of Pods, how they should be
handled when scaling in or out, and most importantly, how to roll out the changes in Pod `spec`. Deployments can be
updated, triggering or not a `rollout`, which is a rolling update that will bring in changes to immutable fields of
`ReplicaSets` (for example, the `selector` field), or propagate changes in a Pod `template`'s container image.

> ⚠️ If the Deployment is using `.spec.strategy.type: Recreate`, no new Pods will coexist with old ones, which means 
> there will be a timeframe with no available Pods, leading to service disruption. This isn't to say `Recreate`
> shouldn't be used, but rather that it serves a purpose that is often not the typical use-case for stateless 
> applications.
 
Since Deployments manage ReplicaSets, the API Server tracks them the same way ReplicaSets track Pods: by using labels to
track ownership, and adding fields to reference who controls what. Because of this, special attention should be put into
carefully selecting labels for `selector`, so ReplicaSet and Pod ownership doesn't result in ownership conflict, leading
to unexpected results.

## Updating through Rollout

> ℹ️ A _rollout_ is triggered every time a change is made to a Deployment's Pod `spec`. Not all changes to a Deployment
> trigger a rollout. Simple updates such as scaling in and out will modify the Deployment, which then trigger Pods to be
> created or deleted, by changing the ReplicaSet's `replica` count. No new ReplicaSet is created just for this.

Earlier, when we attempted to change the image tag for the REDIS ReplicaSet, what we wanted was a mechanism to trigger 
the replacement of the old Pods, with new ones running the updated image we set. With a Deployment, that is achievable, 
and the controller will make sure the new Pods come in without affecting the application's availability.

In this case, for the change of container image, the process unfolds like this:
1. We update the manifest to change the image;
2. The API acknowledges the change, and modifies the object's _desired_ state;
3. Since there's a mismatch to the Deployment state, when it comes to the `spec.template.spec`, the controller triggers
   a rollout;
4. A new ReplicaSet is created, following the updated `spec.template` on the Deployment, and attached to this 
   Deployment;
   1. As the new ReplicaSet is associated, the Deployment controller actually scales down the previous ReplicaSet, as
      the new begins creating its Pods. Scaled down (to `0`) ReplicaSets are kept in the cluster for the 
      `revisionHistoryLimit`, and can't be modified by updating the Deployment (unless a rollback is issued).
      1. Scaling down ReplicaSets like this allows for graceful termination of Pods (which, in turn, allows for 
         application containers to clean up and exit), along with guarantees of Pod availability, as not to disrupt the
         application.
   2. During the rollout, the Deployment actually has two ReplicaSets, but the desired Pod count is obtained by looking
      at the most recent ReplicaSet.
      1. While there might be more Pods associated with the Deployment, from an old ReplicaSet, their Pods do not count
         towards the desired `replicas` calculations, because they're marked for termination.
5. Once the new ReplicaSet has all its Pod available and ready, the old one, which is scaled down to 0, is deleted.
6. The observed state now matches the desired state, and the rollout was successful.

Rollouts can be `paused`, if we need to ensure changes only take effect when we decide so (for example, we noticed that
the change we applied isn't right, and want to halt the update until we push one or more changes to be applied).
- To pause a deployment, we can use `kubectl rollout pause deployment/<name>`. This will **not** affect the currently
deployed application, beyond whatever changes were done before pausing. While paused, any new update is processed and
queued, to be rolled out once the rollout is `resumed`.
  - If the subsequent updates, during a `pause`, change Pod `spec`, it may lead to a _Rollover_, post `resume`.

### Rollover (or RollingUpdate)
A rollover is a special case of a rollout, in that we introduce updates to an already updating Deployment. Rather than 
blocking such changes, the API Server will accept and queue this update, replacing queued actions for the initial update
with the actions needed to be carried out for the latest update, when applicable. Let's go over an example:

Imagine the REDIS cluster, which currently sits on 2 replicas with the `v3` image. Right after we updated the tag to 
`v5`, we noticed that `v4` is actually the tag we want, so we make that change. By the time we applied, the Deployment
was already half-way through the upgrade, with one (old) Pod using `v3`, and a (new) Pod using `v5`, which means there
are two ReplicaSets associated. Now, the controller launches a third ReplicaSet, the one using the `v4` image, and marks
the intermediate one (with `v5` image) for deletion. The running Pods using `v3`and `v5` images, are now scheduled for 
deletion, the other `v5` Pod will be deleted (if not yet scheduled), or terminated (if it had already been scheduled),
and the two new `v4` Pods will be scheduled. All this is done in-flight, and the API won't even wait for a ReplicaSet to
complete bringing its Pods online, before marking it for deletion, so it can be replaced. In the end, the most recent
ReplicaSet prevails, and there's a lot of transient states, consisting of a mix of the three ReplicaSet's Pods.
- During all of this, if the application accepts incoming request, the traffic will be sent to Pods still available.
  This means there is a timeframe where `v3` and `v5` Pods could be serving requests, as well as a timeframe where `v5`
  and `v4` are valid candidates for requests. There's a lot to unpack here, from strategy to networking, which we'll
  cover at a later time.

> ⚠️ **A Deployment's `selector` field is immutable**. There is no higher abstraction that lets us manage Deployments, 
> so that change can be rolled out. While Pod's labels can be changed, as well as ReplicaSet's `selector`, this should 
> be avoided, as it can disrupt applications. Read more about it 
> [here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#label-selector-updates).

### Scaling Deployments

Just as you can scale a ReplicaSet in and out, so can you do so for a Deployment. The interesting part here is that the
Deployment resource allows for a considerable amount of configurations, to customize exactly how to carry out the
scaling actions. This is configured in the `strategy` field of the Deployment `spec`.

While the defaults are typically enough, this is an advanced setting that lets us ensure some QoS for applications, like
always keeping a certain number of Pods always available, regardless of the Deployment's replica count. By using this,
an update can be blocked because it'd bring the count below the threshold we set, potentially degrading the service the
application provides.

By setting `maxSurge` to any number over `0`, we're telling the API Server that, during a rolling update, a Deployment
can have up to `spec.replicas + N`, meaning it'll have up to `N` other Pods available, and potentially receiving 
requests. This is an important setting, as it decreases the amount of change that is introduced to the application, by
trickling in a few Pods at a time. By setting `maxUnavailable`, we're telling the API Server that this Deployment can't
have more than `N` old Pod terminations in-flight. This means that, for the old ReplicaSet, only a few Pods will be
terminated at a time, allowing new ones to replace them. This is important because it's what guarantees that there will
be Pods to continue processing requests, or whatever work they do, during a rolling update. 

If `maxSurge` is `0`, `maxUnavailable` **can't be** `0` either, as it'd block any attempt of updating the Deployment. 
Setting `maxUnavailable` to `0`, and `maxSurge` to `1`, for example, ensures the number of Pods available in the 
Deployment never goes below the desired replica count, and new Pods are brought in, one by one.

## Creating Deployments

If you need to deploy an application, chances are you'll end up using a Deployment. Just like every other resource we've
covered, there's a lot to it that doesn't need to be in the manifest, since the defaults are, generally, enough.

Here's what a very simple Deployment manifest would look like:
````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
````
- This will create an `nginx` application, running `3` Pods, listening on port `80`. Super simple.

Now, there is a lot more we can do. Let's take a look at an example:

<details>
  <summary><strong>Deployment description</strong></summary>

````shell
❯ kubectl --context production --namespace example-app get deployments.apps example-app
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
example-app   1/1     1            1           3y77d
❯ kubectl --context production --namespace example-app describe deployments.apps example-app
Name:               example-app
Namespace:          example-app
CreationTimestamp:  Tue, 08 Oct 2019 21:34:11 +0100
Labels:             SHA=0ef5ea8
                    app=example-app
                    build=example-app-deploy-802
                    type=web
Annotations:        deployment.kubernetes.io/revision: 2547
                    kubectl_plugins-stopped: false
Selector:           app=example-app,type=web
Replicas:           1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:       Recreate
MinReadySeconds:    0
Pod Template:
  Labels:       app=example-app
                type=web
  Annotations:  SHA: 0ef5ea8
                build: jenkins-example-app-deploy-802
                iam.amazonaws.com/role: kube2iam-production-example-app
                kubectl.kubernetes.io/default-container: example-app-web
                kubectl_plugins-restarted: 2022-10-09 11:57:37 -0400
  Containers:
   example-app-web:
    Image:      registry:8080/example-app:0ef5ea8
    Port:       <none>
    Host Port:  <none>
    Command:
      bundle
    Args:
      exec
      rake
      jobs:pump
    Limits:
      cpu:     1
      memory:  512Mi
    Requests:
      cpu:     250m
      memory:  512Mi
    Environment Variables from:
      example-app-0ef5ea8  ConfigMap  Optional: false
      example-app-0ef5ea8  Secret     Optional: false
      web-0ef5ea8   ConfigMap  Optional: false
    Environment:         <none>
    Mounts:
      /etc/secrets from etc-secrets (rw)
  Volumes:
   etc-secrets:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  etc-secrets
    Optional:    false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   example-app-web-7bb644f864 (1/1 replicas created)
Events:          <none>
````
</details>

There's a lot here. But none of this is really new to us:
- There's some `metadata`
  - The Deployment has a number of labels to help automation, of which `app` and `type` are propagated to the Pods.
- The `selector` labels are `app` and `type`, so all of `example-app`'s `web` Pods flock together.
- The Deployment currently has `1` replica `available`, and is at the _desired_ state.
- The `stategy` uses, for updating the Deployment is `Recreate`.
  - Once a change is made, all current Pods are deleted, before new ones are created to replace them.
- The `Pod` this Deployment create is fairly simple, too:
  - Labels match the selector 👍
  - The Pod doesn't listen on any `Port`, so it's safe to assume it isn't expecting requests.
  - It has a single container, `example-app-web`.
    - It runs `bundle exec rake jobs:pump` when it launches
    - It can use, at most, `1` CPU unit (core) and `512`Mi of memory. It starts with `250` milli-CPU, and `512`Mi of
      memory pre-allocated.
    - The container will have `env` variables mounted from a couple of `ConfigMaps` and `Secrets`.
    - The container also maps a single `volume`, mounted to `/etc/secrets`.
- The Pod specifies access to a `volume`, to be accessible by the container, from a `secret`.

# Quick Recap

- Workloads lean heavily on Pods, but since there aren't really cut out to be handled alone, we introduce Sets to manage
  them. These are abstractions for Pod controllers, with different _flavours_:
  - `StatefullSet`: Manage Pods in a ordered, sticky-to-Nodes way. Each Pod gets a unique, serialized identifier name,
    that is kept across rescheduling (which happens always to the same Node, if it's available).
    - StatefulSets are great for applications that need persistent storage, and guarantees of persistence through Pod
      rescheduling, particularly for storage backends.
  - `DaemonSet`: Ensure all _suitable_ Nodes get a replica of the Pod this Set specifies.
    - Particularly relevant for applications that typically are associated with Kubernetes services, or that should be run
      on every instance (observability, security, fleet management, etc.).
  - `ReplicaSet`: The _bread-and-butter_ of stateless application representation in Kubernetes. Provides management of Pod
    replicas, ensuring the number of running Pods match the desired count.
    - Pods are managed, just like in other Sets, by references of who owns them, and are acquired by virtue of `selector`
      labels. ReplicaSets replace Pods when these crash, or are preempted (moved from a Node), but won't replace them if
      the Pod `template` changes.
- While `ReplicaSets` are very popular for stateless applications, they fall short on being able to roll-out changes to
  the Pod `spec`, which is vital for application day-to-day operations.
- To deal with the shortcomings of ReplicaSets, we introduce `Deployments` as a controller of ReplicaSets.
  - Deployments are able to update Pod templates, using a strategy to carry out the update (rolling, or strict **stop
    and swap**) of the Pods, ensuring that the application is _eventually_ updated.
    - By controlling the number of old Pods that persist, and new ones that are introduced, throughout the update, we
      can set the pace of the update, opting for a speedy, or uncompromising strategy. 
  - Deployments are also able to be rolled back, should our update be unsuccessful, and we wish to revert the change.
    They can also be paused, meaning no update is rolled out until the Deployment is resumed.

# What's Next?

Read the [Part 3](./kubernetes-fundamentals_Defining_Workloads_part-3.md) for how to manage workloads that shouldn't be
constantly running, and are expected to complete after they've done what they set out to do.

If you're looking to put into action what you've learned about Deployments, take a look at the
[Lab 1 - Deploying an Application](../../labs/kubernetes/lab1-deploying_an_application.md).
