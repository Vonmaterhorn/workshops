# Kubernetes Fundamentals - How Kubernetes _Works_ (Part 2)

In the previous part we delved a bit into how a Kubernetes cluster works: the constant reconciliation loop of matching
observed state to desired state. While we've only scratched the surface on workloads, and we'll go through that concept
more in depth in later tutorials, it's important to actually know what we're getting out of all these manifests, and see
what actually is happening in a cluster.

For now, let's focus on how to interact with resources and objects in a Kubernetes cluster. 

## Interacting with a Cluster

We've covered the basis on how a Kubernetes cluster comes to be, how it works, and how we can create objects to serve
our needs. From the snippets in the last part of the tutorial, the interaction with the cluster comes in the form of a 
command-line tool called [**kubectl**](https://kubernetes.io/docs/reference/kubectl/).

Kubernetes interaction actually occurs at the API level. In fact, all interactions with Kubernetes are API calls, but
`kubectl` offers a lot of commodity in abstracting these calls behind simple commands, arguments and flags, that what
we'd get from sending HTTP requests, along with the payloads, directly to the API endpoints.

While there are other tools to communicate with the Kubernetes Control Plane (such as [K9S](https://k9scli.io/)), we'll
focus on this tool for the time being, as it's widely regarded as the _de facto_ tool to use when managing Kubernetes
resources and objects.

In order to be able to interact with the API Server, we need to, first, know how to reach it. In fact, while the API
Server endpoint is just an HTTP endpoint, there's a lot more to it than just a web server listening for HTTP requests:

![](https://d33wubrfki0l68.cloudfront.net/673dbafd771491a080c02c6de3fdd41b09623c90/50100/images/docs/admin/access-control-overview.svg)
- From [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)

While we won't go into detail all the security layers, and RBAC capabilities of Kubernetes, there is one thing we need
to go through, when in comes to interacting with a cluster, and that is the `kubeconfig` file.

### The kubeconfig file

First, `kubeconfig` in a moniker for a file that organizes access to any number of Kubernetes cluster. The actual file
is often found in the `~/.kube/config` path, hence the name.

A `kubeconfig` file has the following structure:
- A `clusters` section
  - This is where we configure how to access the API Server endpoint of our cluster(s). It specifies:
    - Connection information on how to contact the API Server for a given cluster: 
      - The CA (certificate authority) information, so it can make sense of the self-signed certificates.
      - The Server endpoint, as a URL.
      - Whether to use TLS verification (`insecure-skip-tls-verify: true|false`).
    - A name, to uniquely identify the cluster. 
- A `users` section
  - This is where authentication to the cluster is specified. It can client certificate data, for cryptographic 
    authentication, _Bearer_ tokens, or username and password credentials. Typically, a certificate is used for 
    authentication in production cluster, while for local cluster, the _Bearer_ is used.
- A `contexts` section
  - This is how Kubernetes keeps track of what information is related to each other, when it comes to access each 
    cluster, by bundling `cluster` and `user` information to create a `context`.
    - Contexts' names have to be unique among them.
  - We can also specify the user's preferred `namespace` here. If not provided, the `default` namespace is set. This
    doesn't limit the user to that namespace, but rather sets it as the pre-defined namespace to use when none is set,
    when interacting with the cluster.
- The `current-context` field
  - This specifies which context to run commands on, by default, if no `context` is passed in the command.
- The typical fields of a Kubernetes resource definition
  - `apiVersion`: is often `v1`.
  - `kind`: in this case, is `Config`.

While this is important to know, as we're going through the gears that make Kubernetes work, the management of this file
is often left to the Kubernetes solution we choose. Users typically never interact directly with the `kubeconfig` file.
The `kubeconfig` handles only the Authentication part. The Authorization is handled by Kubernetes, but we'll visit the
security aspects and Kubernetes resources tied to this concept, at a later time. 

To read more about this, check out 
[Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) 
and
[Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/).

## Using `kubectl` to select the context

When `kubeconfig` has a single cluster configured, the default context is that cluster. If there are multiple, however,
we need to select the right context for which to run commands on. This can be done in two ways:
- Setting the default context: `kubectl config use-context`
- Setting the context on which to run the command: `kubectl get pods --context <context name>`

It's generally a best-practice to set the context on which you'll work most often, instead of adding a `--context`
parameter to each command, as the likelihood of forgetting, and running the command against the wrong context, goes up
with each command you run.

## Using `kubectl` to visualize objects

There are several ways one can observe the API resources, and get objects' information, their states, or their
definitions.

### kubectl explain

To get more information on an API resource, run `kubectl explain <resource>`:

````text
❯ kubectl explain Pods
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind	<string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata	<Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec	<Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

   status	<Object>
     Most recently observed status of the pod. This data may not be up to date.
     Populated by the system. Read-only. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
````

It's useful when setting up a manifest to interact with the resource, for either creating or updating a resource.

### kubectl get

This returns a simplified representation of the object requested. To `get` an object, specify its resource (or `kind`),
its name, and its namespace (if it's not the default):

````text
❯ kubectl get pod hello-world
NAME          READY   STATUS      RESTARTS   AGE
hello-world   0/1     Completed   0          10m
````

### kubectl describe

To get a more detailed state on the object, along with its status and events, use `describe` with the same arguments as
`get`. It's a much more verbose output, useful to troubleshoot or simply to get more detailed information about a
resource.

### kubectl logs

Containers typically output to STDOUT. Kubernetes manages these through the concept of container logs, which are visible
throughout the life of a Pod. To get the logs of a pod, issue `kubectl logs pod <name of the pod>`:

````text
❯ kubectl logs hello-world
hello world!
````

## Using `kubectl` to create, update, or destroy objects

### kubectl apply

The simplest way to create objects in Kubernetes is by writing a manifest, that specifies the object(s) to be created,
and feeding it to `kubectl`. Above, when creating the manifest for the Pod object, a YAML was passed to the tool, using
the `-f` flag. Here's what it does:
````
-f, --filename=[]: that contains the configuration to apply
````
Running `kubectl apply -f <file name>-yaml` will do the following, _if it's a valid YAML or JSON file_:
- Read the _apiVersion_ to know which API to send the request
- Read the _kind_ to figure out which resource to use
- Read the _metadata_ to specify the attributes of the object, such as _name_ and _namespace_
- Read the _spec_ to characterize the object to create or modify
- Create a request to the API with this object
- Pass along to the user any responses from the API, in a customizable output format

The API, upon receiving a valid request, will attempt to process and create or modify the object, against the specified
resource. If there are no impediments, the object will be created, along with a status, in the API. However, if, for any
reason. the API cannot fulfill the request (for e.g., the request references a namespace than doesn't exist, or the user
requesting it doesn't have permissions, or the object `spec` doesn't comply to the resource specification), the API
will error, and the user will be presented with the reason.

The command will often wait for the object to be created before returning to the prompt. There are circumstances that
pay into the wait time, such as amount of resources to be contacted, the amount of objects being applied, the time it
takes for the API and the controllers to run their control-loops. In the case of Cloud resources, objects being modified
in cluster trigger controllers to run their control-loops to validate that the desired state is met, and adjust the
Cloud objects accordingly. In this case, the `kubectl apply` may succeed and return to the prompt, but the state of the
objects may still be transient.

When running `kubectl apply -f -`, instead of a file, the manifest is read from STDIN. This is useful when piping
expression in the command line.

### kubectl create

The `create` command is one of the most powerful commands at a users' disposal, simply because it abstracts most of the
complexity behind creating a manifest, which is creating the correct `spec`. Here's what `create` can do:
````text
Create a resource from a file or from stdin.

 JSON and YAML formats are accepted.

Examples:
  # Create a pod using the data in pod.json
  kubectl create -f ./pod.json

  # Create a pod based on the JSON passed into stdin
  cat pod.json | kubectl create -f -

  # Edit the data in docker-registry.yaml in JSON then create the resource using the edited data
  kubectl create -f docker-registry.yaml --edit -o json

Available Commands:
  clusterrole         Create a cluster role
  clusterrolebinding  Create a cluster role binding for a particular cluster role
  configmap           Create a config map from a local file, directory or literal value
  cronjob             Create a cron job with the specified name
  deployment          Create a deployment with the specified name
  ingress             Create an ingress with the specified name
  job                 Create a job with the specified name
  namespace           Create a namespace with the specified name
  poddisruptionbudget Create a pod disruption budget with the specified name
  priorityclass       Create a priority class with the specified name
  quota               Create a quota with the specified name
  role                Create a role with single rule
  rolebinding         Create a role binding for a particular role or cluster role
  secret              Create a secret using specified subcommand
  service             Create a service using a specified subcommand
  serviceaccount      Create a service account with the specified name
````

To know more about a specific `create` command on a resource, run `kubectl create <resource> --help`

### kubectl delete

Just like `apply` creates or updates an object, `deletes` removes the object from the cluster. It uses the same approach
as stated before, when it comes to specifying the manifests, accepting either a file, or STDIN.

# Quick Recap
- All interaction with a Kubernetes cluster is done through API calls, to the cluster's API Server endpoint.
  - To interact with a cluster, we need to know its CA certificate, and the URL for the API Server.
  - Authentication to the cluster is done by specifying user data, either certificates, tokens or credential sets.
    - The first if typically what production systems use to access their clusters, while tokens are generally what is used in 
      local cluster access.
  - The tools we use are just wrappers around resource API invocations, and some are capable of even generating the 
    payloads to create objects for ourselves.
- Kubernetes should be interacted through use of `kubectl`.
  - Before setting out to interact with a cluster, make sure to set the right `context`.
    - This is handled by the `kubeconfig` file, though manipulation of such file should be done through `kubectl`.
  - `kubectl` offers many commands to abstract the complexity of issuing the corresponding API requests to resources or
  objects.
  - `kubectl` can be used to `apply` Kubernetes manifests, or `create` Kubernetes resources or objects. It can also 
  `delete`, `describe`, and even `explain` them.
  - The output of a Pod's container is store as `log`s, which `kubectl` is able to show.  

# What's Next

You should now read [Defining Workloads - Part 1](./kubernetes-fundamentals_Defining_Workloads_part-1.md).

If you're looking to learn how to deploy your first cluster, take a look at the
[Lab 0 - Setting up a Kubernetes Cluster](../../labs/kubernetes/lab0-setting_up_k8s_cluster.md).
