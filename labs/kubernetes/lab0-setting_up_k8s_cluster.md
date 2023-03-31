# Lab 0 - Setting up a Kubernetes Cluster

## Goal: Get a Kubernetes cluster running, from scratch, in a local setup.

During this exercise we’ll create a Kubernetes cluster running locally in Virtual Machines. This cluster will:
- Run a single Control Plane node
- Run a couple of Worker nodes
- Run some of the most popular addon services

During this exercise, we'll also need the following tools:
- VirtualBox
- Vagrant
- microk8s

This is a very simple exercise, just to set up the cluster, and interact with it to see the core components, using
`kubectl`. All this will be done inside VMs, to better manipulate the environment Kubernetes clusters operate in .

## Step 1 - Create the Virtual Machines

The following steps are necessary to set up the VMs in which we'll actually run the lab. Don't worry too much about what
we're doing here, but if you're curious about Vagrant, and the concept of creating VMs on the fly, read more about it
[here](https://developer.hashicorp.com/vagrant/docs).

1. Install VirtualBox
   - If you're on Intel (`Darwin`) MacBook, download and run
     [this](https://download.virtualbox.org/virtualbox/7.0.4/VirtualBox-7.0.4-154605-OSX.dmg)
   - If you're on M1/M2 (`ARM64`) MacBook, download and run
     [this](https://download.virtualbox.org/virtualbox/7.0.4/VirtualBox-7.0.4_BETA4-154605-macOSArm64.dmg)
   - Check it's installed and using a somewhat recent version
     ````shell
     VBoxManage -v
     7.0.4r154605
     ````
1. Install Vagrant
   ````shell
   brew install vagrant
   ````
   - Should be version >= 2.3.2
     ````shell
     vagrant --version
     Vagrant 2.3.2
     ````
   Update the plugins (otherwise, Vagrant may fail to `init`)
   ````shell
   vagrant plugin update
   ````
1. Create some Ubuntu 22 (JammyJellyfish) VMs with 4Gi of memory, and 4 CPU, to act as our Kubernetes nodes.
   Copy the following to a `Vagrantfile`.
   ~~~shell
   # Vagrant multi-machine sample setup
   Vagrant.configure("2") do |config|
      config.vm.provider "virtualbox" do |v|
         v.memory = 2048
         v.cpus = 2
      end
      config.vm.define :master do |master|
         master.vm.box = "ubuntu/jammy64"
         master.vm.hostname = "master"
         master.vm.network "private_network", ip: "192.168.56.10", virtualbox__intnet: "labnet"
      end
            
      config.vm.define :worker1 do |worker1|
         worker1.vm.box = "ubuntu/jammy64"
         worker1.vm.hostname = "worker1"
         worker1.vm.network "private_network", ip: "192.168.56.11", virtualbox__intnet: "labnet"
      end
            
      config.vm.define :worker2 do |worker2|
         worker2.vm.box = "ubuntu/jammy64"
         worker2.vm.hostname = "worker2"
         worker2.vm.network "private_network", ip: "192.168.56.12", virtualbox__intnet: "labnet"
      end
   end
   ~~~~ 
   - We need to specify the IP addresses, so the nodes can communicate via a shared, private network.
   
   Boot the VM with
   ````shell
   vagrant up
   ````
   - During the boot sequence, you may be prompted to allow VirtualBox (or VBoxManage) access to network and/or disk.
     Please allow it, so Vagrant can finish setting up the VMs.
1. Log into the boxes
   ~~~~shell
   vagrant ssh master
   ~~~~
   - Output should be similar to this:
      ````shell
      Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-53-generic x86_64)
      
      * Documentation:  https://help.ubuntu.com
      * Management:     https://landscape.canonical.com
      * Support:        https://ubuntu.com/advantage
      
      System information as of Fri Dec  2 16:33:42 UTC 2022
      
      System load:  0.029296875       Processes:               133
      Usage of /:   6.2% of 38.70GB   Users logged in:         0
      Memory usage: 5%                IPv4 address for enp0s3: 10.0.2.15
      Swap usage:   0%                IPv4 address for enp0s8: 192.168.56.10
      
      
      11 updates can be applied immediately.
      11 of these updates are standard security updates.
      To see these additional updates run: apt list --upgradable
      
      
      vagrant@master:~$
      logout
      Connection to 127.0.0.1 closed.
      ````
   
   ````shell
   vagrant ssh worker1
   ````
   - Output should be similar to this:
      ````shell
      Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-53-generic x86_64)
      
      * Documentation:  https://help.ubuntu.com
      * Management:     https://landscape.canonical.com
      * Support:        https://ubuntu.com/advantage
      
      System information as of Fri Dec  2 16:34:14 UTC 2022
      
      System load:  0.1826171875      Processes:               129
      Usage of /:   6.2% of 38.70GB   Users logged in:         0
      Memory usage: 5%                IPv4 address for enp0s3: 10.0.2.15
      Swap usage:   0%                IPv4 address for enp0s8: 192.168.56.11
      
      
      14 updates can be applied immediately.
      11 of these updates are standard security updates.
      To see these additional updates run: apt list --upgradable
      
      
      vagrant@worker1:~$
      logout
      Connection to 127.0.0.1 closed.
      `````
        
   ````shell
   vagrant ssh worker2
   ````
   - Output should be similar to this:
      ````shell
      Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-53-generic x86_64)
      
      * Documentation:  https://help.ubuntu.com
      * Management:     https://landscape.canonical.com
      * Support:        https://ubuntu.com/advantage
      
      System information as of Fri Dec  2 16:35:27 UTC 2022
      
      System load:  0.1718758261      Processes:               129
      Usage of /:   6.3% of 38.70GB   Users logged in:         0
      Memory usage: 5%                IPv4 address for enp0s3: 10.0.2.15
      Swap usage:   0%                IPv4 address for enp0s8: 192.168.56.12
      
      
      14 updates can be applied immediately.
      11 of these updates are standard security updates.
      To see these additional updates run: apt list --upgradable
      
      
      vagrant@worker2:~$
      logout
      Connection to 127.0.0.1 closed.
      `````

VMs are all set up, and we're ready to begin setting up our Kubernetes cluster! From now on, all commands should be run
inside the VM hosts, unless instructed otherwise.

## Step 2 - Install `microk8s` on the nodes

[MicroK8s](https://minikube.sigs.k8s.io/docs/start/) is a local Kubernetes solution, great as a starting point to learn
how to manage a Kubernetes system. 
[Kubectl](https://kubernetes.io/docs/reference/kubectl/) is the tool to interact with the Kubernetes Control Plane, and
is the swiss-army-knife of a Kubernetes admin. MicroK8s already packs a `kubectl` for us, so we'll just use that.

1. SSH into the master node
   ````shell
   vagrant ssh master
   ````
1. Install microk8s
   ````shell
   sudo snap install microk8s --classic --channel=1.25
   ````
1. Add the user _vagrant_ to the 'microk8s' group (so we can run it w/o sudo)
   ````shell
   sudo usermod -a -G microk8s vagrant
   sudo chown -f -R vagrant ~/.kube
   newgrp microk8s
   ````
1. Alias kubectl
   ````shell
   echo "alias kubectl='microk8s kubectl'" >> ~/.bash_aliases
   ````
1. Configure kubectl autocomplete
   ````shell
   echo 'source <(kubectl completion bash)' >>~/.bashrc
   exec bash
   ````
   - Now, when typing `kubectl` commands, simply _double tab_ to get suggestions.
      ````shell
      vagrant@ubuntu-jammy:~$ kubectl config
      current-context  delete-context   get-clusters     get-users        set              set-context      unset            view
      delete-cluster   delete-user      get-contexts     rename-context   set-cluster      set-credentials  use-context
      ````
1. Validate the version
   ````shell
   kubectl version
   ````
   - Should output
      ````shell
      WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
      Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.4", GitCommit:"872a965c6c6526caa949f0c6ac028ef7aff3fb78", GitTreeState:"clean", BuildDate:"2022-11-14T22:12:49Z", GoVersion:"go1.19.3", Compiler:"gc", Platform:"linux/amd64"}
      Kustomize Version: v4.5.7
      Server Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.4", GitCommit:"872a965c6c6526caa949f0c6ac028ef7aff3fb78", GitTreeState:"clean", BuildDate:"2022-11-14T22:13:32Z", GoVersion:"go1.19.3", Compiler:"gc", Platform:"linux/amd64"}
      ````
1. Log into each worker node, and repeat steps 2. through 6.
  - It's advised to do this in another terminal, so you don't have to ssh in and out of nodes every time.

When we install microk8s through snap, it'll automatically launch the cluster creation process. As soon as the command
`microk8s status --wait-ready` completes, the cluster is ready. Run `microk8s status` to see how it's configured:

````shell
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
  disabled:
    cert-manager         # (core) Cloud native certificate management
    community            # (core) The community addons repository
    dashboard            # (core) The Kubernetes dashboard
    dns                  # (core) CoreDNS
    gpu                  # (core) Automatic enablement of Nvidia CUDA
    host-access          # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    ingress              # (core) Ingress controller for external access
    kube-ovn             # (core) An advanced network fabric for Kubernetes
    mayastor             # (core) OpenEBS MayaStor
    metallb              # (core) Loadbalancer for your Kubernetes cluster
    metrics-server       # (core) K8s Metrics Server for API access to service metrics
    observability        # (core) A lightweight observability stack for logs, traces and metrics
    prometheus           # (core) Prometheus operator for monitoring and logging
    rbac                 # (core) Role-Based Access Control for authorisation
    registry             # (core) Private image registry exposed on localhost:32000
    storage              # (core) Alias to hostpath-storage add-on, deprecated
````

🚀 Cluster successfully created! For now, only the Control Plane is running, but we need Worker nodes, too. Typically, a 
Kubernetes cluster runs the Control Plane in High Availabilty, meaning more than one node. For consensus and resilience,
it's often run as a group of three, but for this case, one will be enough to get us started.

## Step 3 - Enable RBAC in the cluster

Add the `rbac` [addon](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). This will enable 
**R**ole **B**ased **A**ccess **C**ontrol Auhtorization to the cluster.
> Kubernetes RBAC is a key security control to ensure that cluster users and workloads have only the access to resources
> required to execute their roles. It is important to ensure that, when designing permissions for cluster users, the 
> cluster administrator understands the areas where privilege escalation could occur, to reduce the risk of excessive 
> access leading to security incidents.
- [from RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/). This resources as a lot
of good advice on how to run and operate Kubernetes securely. 

Run the follwing in the **master** host:
````shell
microk8s enable rbac
````
We need to enable this before adding workers, or else, there's a chance some of them fail to become "Ready".

## Step 4 - Add Workers

In microk8s, each node starts as a Control Plane node (its own cluster, basically), and can then join another cluster
as either a Control Plane node, or a Worker node. Run the following in the **master** host:

1. Add name resolution for each node (needed for inter-node communication)
   ````shell
   echo '192.168.56.10 master master' | sudo tee -a /etc/hosts
   echo '192.168.56.11 worker1 worker1' | sudo tee -a /etc/hosts
   echo '192.168.56.12 worker2 worker2' | sudo tee -a /etc/hosts
   ````
   - Here's what the `/etc/hosts` file should look like:
      ````shell
      vagrant@master:~$ cat /etc/hosts
      127.0.0.1	localhost
      
      # The following lines are desirable for IPv6 capable hosts
      ::1	ip6-localhost	ip6-loopback
      fe00::0	ip6-localnet
      ff00::0	ip6-mcastprefix
      ff02::1	ip6-allnodes
      ff02::2	ip6-allrouters
      ff02::3	ip6-allhosts
      127.0.1.1	ubuntu-jammy	ubuntu-jammy
      
      127.0.2.1 master master
      192.168.56.10 master master
      192.168.56.10 worker1 worker1
      192.168.56.10 worker2 worker2
      ````
1. Generate the token for a new node to join the cluster as a worker
   ~~~~shell
   vagrant@master:~$ microk8s add-node
   From the node you wish to join to this cluster, run the following:
   microk8s join 10.0.2.15:25000/d4345e48704cca768ee2a226e2a3b81f/98e6b6a562c8
   
   Use the '--worker' flag to join a node as a worker not running the control plane, eg:
   microk8s join 10.0.2.15:25000/d4345e48704cca768ee2a226e2a3b81f/98e6b6a562c8 --worker
   
   If the node you are adding is not reachable through the default interface you can use one of the following:
   microk8s join 10.0.2.15:25000/d4345e48704cca768ee2a226e2a3b81f/98e6b6a562c8
   microk8s join 192.168.56.10:25000/d4345e48704cca768ee2a226e2a3b81f/98e6b6a562c8
   ~~~~
     - Use the last command, with the `192.168.56.10` ip, and append `--worker`, when running on the worker node
1. Run the following in **worker1**, to have it join the cluster
   `````shell
   vagrant@worker1:~$ microk8s join 192.168.56.10:25000/d4345e48704cca768ee2a226e2a3b81f/98e6b6a562c8 --worker
   Contacting cluster at 192.168.56.10
   
   The node has joined the cluster and will appear in the nodes list in a few seconds.
   
   This worker node gets automatically configured with the API server endpoints.
   If the API servers are behind a loadbalancer please set the '--refresh-interval' to '0s' in:
   /var/snap/microk8s/current/args/apiserver-proxy
   and replace the API server endpoints with the one provided by the loadbalancer in:
   /var/snap/microk8s/current/args/traefik/provider.yaml
   `````
1. Repeat steps 2 and 3 in **worker2** to have it join the cluster   
   ````shell
   vagrant@master:~$ microk8s add-node
   From the node you wish to join to this cluster, run the following:
   microk8s join 10.0.2.15:25000/a110dc71a934f93bfce2143ec52691d6/98e6b6a562c8
   
   Use the '--worker' flag to join a node as a worker not running the control plane, eg:
   microk8s join 10.0.2.15:25000/a110dc71a934f93bfce2143ec52691d6/98e6b6a562c8 --worker
   
   If the node you are adding is not reachable through the default interface you can use one of the following:
   microk8s join 10.0.2.15:25000/a110dc71a934f93bfce2143ec52691d6/98e6b6a562c8
   microk8s join 192.168.56.10:25000/a110dc71a934f93bfce2143ec52691d6/98e6b6a562c8
   
   vagrant@worker2:~$ microk8s join 192.168.56.10:25000/a110dc71a934f93bfce2143ec52691d6/98e6b6a562c8 --worker
   Contacting cluster at 192.168.56.10
   
   The node has joined the cluster and will appear in the nodes list in a few seconds.
   
   This worker node gets automatically configured with the API server endpoints.
   If the API servers are behind a loadbalancer please set the '--refresh-interval' to '0s' in:
   /var/snap/microk8s/current/args/apiserver-proxy
   and replace the API server endpoints with the one provided by the loadbalancer in:
   /var/snap/microk8s/current/args/traefik/provider.yaml

   `````
1. Confirm worker1 and worker2 have successfully joined the cluster, by running `microk8s status` on each of them:
   ````shell
   vagrant@worker1:~$ microk8s status
   This MicroK8s deployment is acting as a node in a cluster.
   Please use the control plane node.
   
   vagrant@worker2:~$ microk8s status
   This MicroK8s deployment is acting as a node in a cluster.
   Please use the control plane node.
   ````
   - If you see one of the workers not outputting the right status, you may have forgotten to append the `--worker` to 
   the `join` command. No problem! Simply run the following in the master node to remove that node, and join again:
     ````shell
     kubectl delete node <worker1|worker2>
     ````
1. Use `kubectl get nodes` to see the Kubernetes cluster nodes
   ````shell
   vagrant@master:~$ kubectl get nodes
   NAME      STATUS   ROLES    AGE     VERSION
   worker1   Ready    <none>   9m10s   v1.25.4
   worker2   Ready    <none>   8m53s   v1.25.4
   master    Ready    <none>   18h     v1.25.4
   ````
   - Don't worry about the ROLES being set to \<none>, the nodes just don't have the right labels..._yet!_

## Step 5 - Finishing Touches

The cluster is up and running, but we want to add some [addons](https://microk8s.io/docs/addons) before we start using 
it. Do the following in the master host:

1. Add the `dns` [addon](https://microk8s.io/docs/addon-dns)
   
   This will install CoreDNS, which allow us to use DNS for service discovery, one of the key features for inter-service
   communication.
   ````shell
   microk8s enable dns
   ````
1. Add the `hostpath-storage` [addon](https://microk8s.io/docs/addon-hostpath-storage)
   
   By default, each container uses the disk on the node it's scheduled in. Since containers can be restarted and moved
   through nodes, the storage is considered ephemeral. This addon allows us to request persistent storage for a Pod.
   ````shell
   microk8s enable hostpath-storage
   ````
1. Add the `metrics-server` [addon](https://github.com/kubernetes-sigs/metrics-server)
   
   This will allow the cluster to be aware of the resources it's using, and introduce some QoL information when checking
   for resource usage.
   ````shell
   microk8s enable metrics-server
   ````
   
As it is, the worker and master nodes aren't labeled as such. While this isn't a problem, it's better to have them
labeled accordingly, so it's easier for us to know which is which (even if the node name is indicative). Run the 
following in the **master** node:
````shell
vagrant@master:~$ kubectl label node master "node-role.kubernetes.io/master="
node/master labeled
vagrant@master:~$ kubectl label node worker1 "node-role.kubernetes.io/worker="
node/worker1 labeled
vagrant@master:~$ kubectl label node worker2 "node-role.kubernetes.io/worker="
node/worker2 labeled
````

🎉 Congratulations, we now have a Kubernetes cluster working on three nodes: a master, and two workers! ⎈

Here's what the cluster should look like:

````shell
vagrant@master:~$ kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
master    Ready    master   19h     v1.25.4
worker1   Ready    worker   2m51s   v1.25.4
worker2   Ready    worker   7m12s   v1.25.4

vagrant@master:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   hostpath-provisioner-766849dd9d-hz95z      1/1     Running   0          42m
kube-system   coredns-d489fb88-hj6dv                     1/1     Running   0          45m
kube-system   calico-kube-controllers-55f954d4db-229q6   1/1     Running   0          19h
kube-system   calico-node-bs6ns                          1/1     Running   0          19h
kube-system   metrics-server-6b6844c455-w95pg            1/1     Running   0          30m
kube-system   calico-node-7drq4                          1/1     Running   0          7m19s
kube-system   calico-node-6wd99                          1/1     Running   0          7m43s
````

Should you wish to save resources, you can now logoff from the VMs, and suspend them until you need them again:
````shell
vagrant suspend master worker1 worker2
````

Lab completed! ✅

# Troubleshoot

On some occasions, a worker node may join, but fail to set up its network (the calico Pod) correctly, or be unable to
contact the API due to TLS handshake failures. If that happens, remove the node from the cluster (using 
`sudo microk8s leave`), and re-add it (using the `microk8s add-node` in the master, and then `microk8s join` in the 
worker). If it's still not working, usually destroying the VM and recreating it(`vagrant destroy <worker1|worker2>` and
`vagrant up <worker1|worker2>`) should fix it.
  - This is often due to certificate issues when coming from `suspend`, after network changes. While the 
    [troubleshooting page](https://microk8s.io/docs/troubleshooting) hints at only needing to reissue the certificate,
    that solution doesn't seem to work.

---

Upon resuming VMs, sometimes the cluster may seem fine, but the network overlay (`calico`) may present some errors like
"[CNI plugin: error getting ClusterInformation: connection is unauthorized: Unauthorized](https://github.com/projectcalico/calico/issues/5712)",
visible by running `kubectl describe <resource> <object-name>`. If this happens, the best course of action is to:
1. Stop all VM: Shutdown, either through the VirtualBox GUI, or in the host terminal (`sudo shutdown -h now`)
2. Start all VMs: Click _Start_ on the VirtualBox GUI, preferably with 
_[Detachable Start](https://superuser.com/questions/375316/closing-gui-session-while-running-a-virtual-machine-virtual-box)_,
   so you can close the terminal window, and return to the terminal via `vagrant ssh <host>`.

After this, you should be able to use the cluster normally.
