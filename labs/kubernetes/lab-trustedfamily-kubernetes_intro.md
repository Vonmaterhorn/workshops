# Kubernetes Tutorial for Beginners

## What is Kubernetes?

Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.
The _default_ ([not officially, but widely considered to be](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker))
container runtime is `containerd`, which replaced `docker` a while back.

Kubernetes is a great platform if you need:

- **Service Discovery**: Through DNS, services can find each other using a standarized naming convention. This is particularly interesting for internal communication,
  but also for finding which services exist in a cluster (via the SRV record, for example).
- **Traffic Ingress Management**: Kubernetes is capable of processing external traffic and setting up rules on how to serve it to workloads within the cluster.
- **Load Balancing**: Kubernetes is capable of routing traffic meant for a workload to all containers intelligently, ensuring requests are sent to containers
  that can serve them.
- **Storage Orchestration**: Kubernetes can use a wide array of solutions for storage, ranging from built-in, local drivers, to third-party solutions running in the cloud or within the cluster. Workloads can decide on which storage to use from the optons available, and that storage can be ephemeral or persistent.
- **Automated rollouts and rollbacks**: You can describe the desired state for your deployed containers using Kubernetes, and it can change the actual state to the desired state at a controlled rate. For example, you can automate Kubernetes to create new containers for your deployment, remove existing containers and adopt all their resources to the new container.
- **Automatic bin packing**: You provide Kubernetes with a cluster of nodes that it can use to run containerized tasks. You tell Kubernetes how much CPU and memory (RAM) each container needs. Kubernetes can fit containers onto your nodes to make the best use of your resources.
- **Self-healing**: Kubernetes restarts containers that fail, replaces containers, kills containers that don't respond to your user-defined health check, and doesn't advertise them to clients until they are ready to serve.
- **Secret and configuration management**: Kubernetes lets you store and manage sensitive information, such as passwords, OAuth tokens, and SSH keys. You can deploy and update secrets and application configuration without rebuilding your container images, and without exposing secrets in your stack configuration.

## What Kubernetes is not

Kubernetes isn't the solution for all runtime needs. There's a considerable management and cognitive overhead to Kubernetes infrastructure that not
all organizations are capable of handling. Kubernetes basic concepts, while simple from an overview, are limited on what one can achieve from a platform
perspective. The more advanced concepts can become extremely complex, be that in usage, management, or design perspectives.

Kubernetes is state-oriented API. This means it's a platform where every resource relates to an API (and thus a definition), has a current and desired state,
and is constantly checked from the former against the latter. Designs and processes that don't conform or adapt to this concept have a hard time translating
well into Kubernetes.

---

## Key Concepts and Tools

Here are some essential Kubernetes concepts to understand:

1. **Node**: A machine (physical or virtual) in the Kubernetes cluster that runs Pods. Can be responsible for `control-plane` and/or workload scheduling (`worker`).
2. **Cluster**: A set of nodes managed by Kubernetes.
3. **Pod**: The smallest deployable unit in Kubernetes, which can contain one or more containers.
   This is the primary resource for workload definition, and configure how a container runs within the ecosystem.
4. **Deployment**: A resource that defines how to deploy and manage Pods. Defines lifcyle rules for Pods, how a collection of Pods (`ReplicaSet`) behaves.
5. **Service**: A resource that exposes your application to the network, working as the primary definition of connectability.

To create the cluster, we're using [Colima](https://github.com/abiosoft/colima), a tool that sets up container runtimes in Linux Virtual Machines. In fact, Colima
is built on top of [Lima](https://github.com/lima-vm/lima), which is the technology that creates and manages instances of whatever architecture and OS on a host, akin to a virtualizer. Colima is then responsible for setting up the containerization platform, such as `docker` or `containerd`. In this context, it is also
responsible for setting the Kubernetes environment, using [K3S](https://k3s.io/).

To interact and manage the cluster and its workloads, we're using a set of standard-use tools:

1. **Kubectl**: Kubernetes provided command line tool for communicating with a Kubernetes cluster's control plane, using the Kubernetes API.
2. **Helm**: The package manager for Kubernetes, Helm serves as both the technology that allows one to create _charts_, that define applications within the ecosystem; and a repository for those _charts_, to install/uninstall and upgrade such applications.

---

## Step 0: Create the Working Directory

Choose a directory somewhere on your filesystem to run this workshop from (for example, `~/Documents/tf-workshop`). The workshop code assume everything is
run from that directory.

## Step 1: Creating and Setting Up Kubernetes Cluster

1. Install `colima`, `kubectl`, and `helm`:

   ```bash
   brew install lima lima-additional-guestagents colima kubectl helm
   cat > ~/.colima/_templates/tf-workshop.yaml <<EOF
   cpu: 4
   memory: 8
   runtime: containerd
   kubernetes:
     enabled: true
     version: v1.31.2+k3s1
     k3sArgs:
       - --disable=traefik
   network:
     dns:
       - 8.8.8.8
       - 1.1.1.1
     dnsHosts:
       host.docker.internal: host.lima.internal
     hostAddresses: true
   vmType: vz
   rosetta: true
   nestedVirtualization: true
   provision:
     - mode: system
       script: |
         apt update && apt install vim yq -y
         mkdir -p /etc/containerd/certs.d/harbor.lvh.me:30080
         containerd config default > /etc/containerd/config.toml
         tomlq -t '.plugins."io.containerd.cri.v1.images".registry.config_path="/etc/containerd/certs.d"' /etc/containerd/config.toml > /etc/containerd/config.toml.tmp && mv /etc/containerd/config.toml.tmp /etc/containerd/config.toml
         cat > /etc/containerd/certs.d/harbor.lvh.me:30080/hosts.toml <<EOF
         server = "http://harbor.lvh.me:30080"

         [host."http://harbor.lvh.me:30080"]
            capabilities = ["pull", "resolve", "push"]
            insecure_skip_verify = true
         EOF
   EOF
   ```

   - The `provision` section of the configuration includes some setup for a container registry that we'll use later on.

2. Validate the configuration:

   ```bash
   colima template
   ```

   - Exit the edit with `:q` and check if there's any error messages:

     ```bash
     colima template
     INFO[0000] editing in vim
     FATA[0007] error in template: could not load config from file: yaml: line 183: mapping values are not allowed in this context
     ```

     - In this case, there's a wrongly idented line 183 in the YAML file. Fix it, and rerun the `template` command.

3. Start Colima to create the instance, along with the Kubernetes cluster

   ```bash
   colima start --profile=tf-workshop
   ```

4. Verify your cluster is running:

   ```bash
   colima ls

   kubectl get nodes
   ```

   - Should output something like:

     ```bash
     NAME     STATUS   ROLES                  AGE   VERSION
     colima   Ready    control-plane,master   20s   v1.31.2+k3s1
     ```

---

## Step 2: Deploying Basic Cluster Services

Kubernetes clusters created with `colima` already come with basic features, such as DNS. But we also Ingress Controllers, to allow traffic into the cluster, and Observability Services for the workloads running in the cluster.

1. Install the `traefik` Ingress Contoller:

   ```bash
   helm repo add traefik https://helm.traefik.io/traefik --force-update
   helm repo update
   helm upgrade --install traefik traefik/traefik --create-namespace --namespace traefik --set service.type=NodePort --set ingressRoute.dashboard.enabled=true --set ports.web.nodePort=30080 --set ports.websecure.nodePort=30443 --set ports.traefik.expose.default=true --set logs.access.enabled=true
   cat > traefik-ingress.yaml <<EOF
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
   labels:
      app: traefik-dashboard
   name: traefik-dashboard
   namespace: traefik
   spec:
   ingressClassName: traefik
   rules:
   - host: traefik.lvh.me
      http:
         paths:
         - backend:
            service:
               name: traefik
               port:
               name: traefik
         path: /
         pathType: Prefix
   EOF
   echo "## - Traefik Dashboard URL: http://traefik.lvh.me:30080/dashboard/ - ##"
   ```

   With `traefik` installed, we now have a way of routing traffic into the cluster, via ports `30080` (for HTTP) and `30443` (for HTTPS). This setup will funnel
   all requests to services within the cluster through this application, just like NGINX. This is a very powerful tool for networking, as it allows us to setup
   other applications' services in a very simple and straightforward way.

2. Install the `Prometheus` Observability stack:

   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts --force-update
   helm repo update
   cat > prometheus.yaml <<EOF
   alertmanager:
     ingress:
       enabled: true
       hosts:
         - alertmanager.lvh.me
   grafana:
     additionalDataSources:
       - name: Loki
         type: loki
         access: proxy
         url: http://loki-gateway
         isDefault: false
     ingress:
       enabled: true
       hosts:
         - grafana.lvh.me
   prometheus:
     ingress:
       enabled: true
       hosts:
         - prometheus.lvh.me
     prometheusSpec:
       scrapeClasses:
         - default: true
           name: cluster-relabeling
           relabelings:
             - sourceLabels: [ __name__ ]
               regex: (.*)
               targetLabel: cluster
               replacement: my-cluster
               action: replace
   EOF

   helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
   --create-namespace \
   --namespace monitoring \
   --values prometheus.yaml

   kubectl wait --for=condition=Ready pod/$(kubectl get pods -n monitoring --selector 'app.kubernetes.io/name=grafana' -o jsonpath="{.items[].metadata.name}") --namespace monitoring
   ```

   [Prometheus](https://prometheus.io/docs/introduction/overview/) is the industry-standard monitoring and alerting system. With this installed, we're capable
   of harvesting metrics from the cluster (resource consumption, system health and usage, etc) and from applications (by scrapping metric endpoints). Our setup
   has three components:

   - `prometheus`: Time-Series database for metrics, with PromQL (query language) to process the metrics.
   - `alertmanager`: An alarm system built on top of Prometheus, that allows us to define thresholds for triggering notifications to internal or external systems.
   - `grafana`: A feature-complete monitoring system, [Grafana](https://grafana.com/) integrates with Prometheus data to enables us to generate dashboards and visualizations from Prometheus and other datasources.

3. Add the `Loki` to the Observability stack, for logging:

   ```bash
   helm repo add grafana https://grafana.github.io/helm-charts --force-update
   helm repo update

   cat > loki-values.yaml <<EOF
   loki:
     image:
       tag: 3.5.1
     auth_enabled: false
     commonConfig:
       replication_factor: 1
     schemaConfig:
       configs:
         - from: "2024-04-01"
           store: tsdb
           object_store: s3
           schema: v13
           index:
             prefix: loki_index_
             period: 24h
     pattern_ingester:
         enabled: true
     limits_config:
       allow_structured_metadata: true
       volume_enabled: true
     ruler:
       enable_api: true

   chunksCache:
     allocatedMemory: 512

   resultsCache:
     allocatedMemory: 512

   minio:
     enabled: true

   deploymentMode: SingleBinary

   singleBinary:
     replicas: 1

   lokiCanary:
     enabled: false
   test:
     enabled: false

   backend:
     replicas: 0
   read:
     replicas: 0
   write:
     replicas: 0

   ingester:
     replicas: 0
   querier:
     replicas: 0
   queryFrontend:
     replicas: 0
   queryScheduler:
     replicas: 0
   distributor:
     replicas: 0
   compactor:
     replicas: 0
   indexGateway:
     replicas: 0
   bloomCompactor:
     replicas: 0
   bloomGateway:
     replicas: 0
   EOF

   helm upgrade --install loki grafana/loki --create-namespace --namespace monitoring -f loki-values.yaml
   helm upgrade --install promtail grafana/promtail --create-namespace --namespace monitoring
   kubectl wait --for=condition=Ready pod/$(kubectl get pods -n monitoring --selector 'app.kubernetes.io/name=loki' -o jsonpath="{.items[].metadata.name}") --namespace monitoring --timeout 5m
   ```

To complete the observability stack, we add [Loki](https://grafana.com/oss/loki/) and [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/) as the log facility/aggregator and log shipper (respectively). Loki will integrate with Grafana as a datasource, to provide visibility into our logs.

---

## Step 3: Deploying a Container Registry

To use container images within the Kubernetes cluster, they need to be hosted somewhere the cluster has access to. Typically, that means either public or private
registries, such as Docker Hub, Quay.io, GitHub Container Registry, Amazon ECR, etc. For our cluster, we want to be able to run our own container images without
needing to releasing them to these services, which means we'll host our own registry.

For this, we'll use [Harbor](https://goharbor.io/), as it provides the registry and a good UI to interact with it (and also Trivy container scans).

1. Install the Harbor helm chart:

   ```bash
   helm repo add harbor https://helm.goharbor.io  --force-update
   helm repo update
   helm upgrade --install harbor harbor/harbor --create-namespace --namespace harbor --set expose.ingress.className=traefik --set externalURL="http://harbor.lvh.me:30080" --set expose.ingress.hosts.core=harbor.lvh.me
   kubectl wait --for=condition=Ready pod/$(kubectl get pods -n harbor --selector component=core -o jsonpath="{.items[].metadata.name}") --namespace harbor
   ```

2. Create a Harbor project, to host the images:

   ```bash
   curl -X 'POST' \
      'http://harbor.lvh.me:30080/api/v2.0/projects' \
      -H 'accept: application/json' \
      -H 'X-Resource-Name-In-Location: false' \
      -H 'authorization: Basic YWRtaW46SGFyYm9yMTIzNDU=' \
      -H 'Content-Type: application/json' \
      -d '{
      "project_name": "tf-workshop",
      "public": true
   }'
   ```

   - The username and password in the `authorization` header is `admin` and `harbor12345`, respectively.

Harbor web UI is accessible via <http://harbor.lvh.me:30080> with the credentials `admin` and `Harbor12345` (username and password, respectively).

---

## Step 4: Building and Releasing a Container Image

We're now ready to start building and hosting our own container images. Instead of using `docker`, we'll use [nerdctl](https://github.com/containerd/nerdctl/tree/main),
a CLI to interact with `containerd`. The rest of the process is exactly like Docker's.

We'll start off with a simple application, a wiki written in Go, from Golang's official tutorial page: <https://go.dev/doc/articles/wiki/>.

1. Install `nerdctl`:

   ```bash
   colima nerdctl install
   ```

2. Login to Harbor with containerd's `nerdctl`:

   ```bash
   nerdctl login harbor.lvh.me:30080 --username admin --password Harbor12345
   ```

   - This allows us to push images to the registry from our terminal.

3. Download the source code:

   ```bash
   mkdir -p gowiki

   curl -so gowiki/wiki.go "https://go.dev/doc/articles/wiki/final.go?m=text"
   ```

4. Create the wiki pages:

   ```bash
   cat > gowiki/edit.html <<EOF
   <h1>Editing {{.Title}}</h1>

   <form action="/save/{{.Title}}" method="POST">
   <div><textarea name="body" rows="20" cols="80">{{printf "%s" .Body}}</textarea></div>
   <div><input type="submit" value="Save"></div>
   </form>
   EOF

   cat > gowiki/edit.html <<EOF
   <h1>Editing {{.Title}}</h1>

   <form action="/save/{{.Title}}" method="POST">
   <div><textarea name="body" rows="20" cols="80">{{printf "%s" .Body}}</textarea></div>
   <div><input type="submit" value="Save"></div>
   </form>
   EOF

   cat > gowiki/view.html <<EOF
   <h1>{{.Title}}</h1>

   <p>[<a href="/edit/{{.Title}}">edit</a>]</p>

   <div>{{printf "%s" .Body}}</div>
   EOF
   ```

5. Create the Containerfile file (equivalent to Dockerfile):

   ```bash
   cat > Containerfile <<EOF
   FROM golang:1.19.4

   WORKDIR /app

   COPY gowiki .

   RUN go mod init gowiki/wiki && \
      go build wiki.go
   EOF
   ```

   - The filesystem tree should be:

     ```bash
     find .
     .
     ./traefik-ingress.yaml
     ./loki-values.yaml
     ./Containerfile
     ./gowiki
     ./gowiki/edit.html
     ./gowiki/view.html
     ./gowiki/wiki.go
     ```

6. Build and Push the image to the registry:

   ```bash
   nerdctl build -t harbor.lvh.me:30080/tf-workshop/gowiki:latest .

   nerdctl push harbor.lvh.me:30080/tf-workshop/gowiki:latest
   ```

The image is now available on our registry. Access the [registry](http://harbor.lvh.me:30080/harbor/projects/2/repositories) with the `admin` credentials to see
the image we've just pushed.

---

## Step 5: Deploying the Application in Kubernetes

To deploy the application in Kubernetes, we need to define the `Deployment` object, which manages the application's container lifecycle, along with all the definitions
that make the container behave the way we wish it to.

1. Write the following manifest to `deployment.yaml`:

   ```bash
   cat > deployment.yaml <<EOF
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: gowiki
     namespace: gowiki
     labels:
        app: gowiki
        type: web
   spec:
      selector:
         matchLabels:
            app: gowiki
      template:
         metadata:
            name: gowiki-web
            namespace: gowiki
            labels:
               app: gowiki
               type: web
         spec:
            containers:
              - name: gowiki
                command:
                  - ./wiki
                image: harbor.lvh.me:30080/tf-workshop/gowiki:latest
                ports:
                  - containerPort: 8080
                    name: http
   EOF
   ```

2. Create and View the application's namespace:

   ```bash
   kubectl create namespace gowiki

   kubectl get namespaces gowiki
   ```

3. Deploy the application to the cluster:

   ```bash
   kubectl apply -f deployment.yaml
   ```

4. Check the status of the application's Pods:

   ```bash
   kubectl get deployments -n gowiki
   ```

   - You should see the following:

     ```bash
     NAME     READY   UP-TO-DATE   AVAILABLE   AGE
     gowiki   1/1     1            1           8s
     ```

---

## Step 6: Exposing Your Application

The application is now deployed in the `gowiki` namespace of our cluster. Since this is a web application, we want to access it via a browser. Before we can do that, we need to expose it, through the `Service` Kubernetes resource.

1. Expose the deployment as a service:

   ```bash
   kubectl expose deployment --namespace gowiki gowiki --name gowiki-web --port 80 --target-port 8080
   ```

2. Get the service details:

   ```bash
   kubectl get services --namespace gowiki
   ```

   - You should see:

     ```bash
     NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
     gowiki-web   ClusterIP   10.43.204.149   <none>        80/TCP    2m24s
     ```

3. Accessing your application via `port-forward`:

   - This is the simplest way to access our application from a browser:

     ```bash
     kubectl port-forward --namespace gowiki services/gowiki-web 8888:80 &
     ```

     - You'll see the following output:

       ```bash
       Forwarding from [::1]:8888 -> 8080
       ```

     - To kill the background process run: `kill $!`

Now, you can access `http://localhost:8888/view/trustedfamily` in your browser and start creating Wiki articles. 🎉

---

## Step 7: Cleaning Up

1. Delete the service:

   ```bash
   kubectl delete service gowiki-web
   ```

2. Delete the deployment:

   ```bash
   kubectl delete deployment gowiki
   ```

3. Stop the Colima machine to save resource:

   ```bash
   colima stop
   ```

---

## Next Steps

- Learn about [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) and [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
  for managing configuration and sensible data within Kubernetes workloads.
- Explore [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) for stateful applications.
- Look into [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) to see how to configure traffic ingress rules, like FQDN matching.
- Check [ArtifactHub](https://artifacthub.io/) as the Docker Hub for charts you might to install in the cluster with `helm`.

Congratulations! You've completed your first Kubernetes tutorial. 🎉
