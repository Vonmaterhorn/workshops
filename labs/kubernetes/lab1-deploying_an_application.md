# Lab 1 - Deploying an Application

## Goal: Create an application, and deploy it to Kubernetes

During this exercise we’ll create a Kubernetes `Deployment`, and have an application running in our local Kubernetes
cluster. During this exercise, we'll:
- Enable a local container registry, through `microk8s`
- Push an application's container image to our local, private registry
- Write the Deployment manifest
- Deploy the application to our cluster
- Interact with the application, in various ways

During this exercise, we'll use the following tools:
- kubectl
- podman

# Step 1 - Enable the Container Registry

Another of `microk8s` many features is having a container registry readily available as an addon. This is helpful for
local development, as you're creating applications, so you don't need to push to public registries, or cloud registries,
such as ECR, which isn't available/intended for your local Kubernetes cluster to use.

To enable the registry, simply run the following:

````shell
vagrant@worker1:~$ microk8s enable registry
Infer repository core for addon registry
Infer repository core for addon hostpath-storage
Addon core/hostpath-storage is already enabled
The registry will be created with the size of 20Gi.
Default storage class will be used.
namespace/container-registry created
persistentvolumeclaim/registry-claim created
deployment.apps/registry created
service/registry created
configmap/local-registry-hosting configured
````

After this, there will be a new `Deployment` in the cluster, for the _registry_:

````shell
vagrant@worker1:~$ kubectl get Deployments -n container-registry
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
registry   1/1     1            1           2m
````

It's available via HTTPS, on port `32000`:
````shell
vagrant@worker1:~$ kubectl --namespace container-registry get svc
NAME       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
registry   NodePort   10.152.183.215   <none>        5000:32000/TCP   3m
````
- We can access the registry via `curl`:
  ````shell
  vagrant@worker1:~$ curl -I localhost:32000
  HTTP/1.1 200 OK
  Cache-Control: no-cache
  Date: Fri, 30 Dec 2022 15:44:04 GMT
  ````

Read more about the registry addon in the [official microk8s documentation](https://microk8s.io/docs/registry-built-in).
- Mind that, since we're using `containerd` instead of `docker` as our container runtime, listing and interacting with
  containers and images should be done with `microk8s ctr`.  Read more about containerd cli commands and options
  [here](https://github.com/projectatomic/containerd/blob/master/docs/cli.md).
- Since containerd doesn't handle container image creation, we'll use Podman for that purpose. This is covered in the
  next step.

# Step 2 - Build Container Images

Since we'll want to build our own container images, and push them to our local registry, we'll need something other than
`docker` to build and push the images, since `containerd` doesn't provide such utilities. We'll use
[podman](https://docs.podman.io/en/latest/index.html), as it's a simple, daemon-less, Linux tool to build containers
(among other things), using the OCI spec.
- To install `podman`, simply run `sudo apt-get -y install podman`

Let's start with a simple image, one that we can use to interact with URLs inside the cluster: `curl`.
````dockerfile
FROM ubuntu:jammy

RUN apt-get update &&\
apt-get install curl -y
````
- Save this to a file named `Containerimage`. While Docker reserved the filename `Dockerimage`, other OCI tools opt to
  use a more generic name, such as `Containerimage`. Podman accepts both.

Now, to build and push it to our registry:
- Start by building and tagging the image. Since our registry isn't listening on a standard port, we need to tag the
  image with the correct URL format, so we can reference it in our manifests.
  ````shell
  vagrant@worker1:~$ podman image build -t localhost:32000/curl:latest .
  ````
  <details>

  ```shell
  STEP 1/2: FROM ubuntu:jammy
  STEP 2/2: RUN apt-get update &&    apt-get install curl -y
  Get:1 http://archive.ubuntu.com/ubuntu jammy InRelease [270 kB]
  Get:2 http://security.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
  Get:3 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [114 kB]
  Get:4 http://archive.ubuntu.com/ubuntu jammy-backports InRelease [99.8 kB]
  Get:5 http://archive.ubuntu.com/ubuntu jammy/main amd64 Packages [1792 kB]
  Get:6 http://archive.ubuntu.com/ubuntu jammy/universe amd64 Packages [17.5 MB]
  Get:7 http://archive.ubuntu.com/ubuntu jammy/multiverse amd64 Packages [266 kB]
  Get:8 http://archive.ubuntu.com/ubuntu jammy/restricted amd64 Packages [164 kB]
  Get:9 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [970 kB]
  Get:10 http://security.ubuntu.com/ubuntu jammy-security/restricted amd64 Packages [593 kB]
  Get:11 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [972 kB]
  Get:12 http://archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 Packages [641 kB]
  Get:13 http://archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 Packages [8150 B]
  Get:14 http://archive.ubuntu.com/ubuntu jammy-backports/main amd64 Packages [3520 B]
  Get:15 http://archive.ubuntu.com/ubuntu jammy-backports/universe amd64 Packages [7291 B]
  Get:16 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 Packages [781 kB]
  Get:17 http://security.ubuntu.com/ubuntu jammy-security/main amd64 Packages [667 kB]
  Get:18 http://security.ubuntu.com/ubuntu jammy-security/multiverse amd64 Packages [4732 B]
  Fetched 24.9 MB in 2s (10.7 MB/s)
  Reading package lists...
  Reading package lists...
  Building dependency tree...
  Reading state information...
  The following additional packages will be installed:
    ca-certificates libbrotli1 libcurl4 libldap-2.5-0 libldap-common
    libnghttp2-14 libpsl5 librtmp1 libsasl2-2 libsasl2-modules
    libsasl2-modules-db libssh-4 openssl publicsuffix
  Suggested packages:
    libsasl2-modules-gssapi-mit | libsasl2-modules-gssapi-heimdal
    libsasl2-modules-ldap libsasl2-modules-otp libsasl2-modules-sql
  The following NEW packages will be installed:
    ca-certificates curl libbrotli1 libcurl4 libldap-2.5-0 libldap-common
    libnghttp2-14 libpsl5 librtmp1 libsasl2-2 libsasl2-modules
    libsasl2-modules-db libssh-4 openssl publicsuffix
  0 upgraded, 15 newly installed, 0 to remove and 0 not upgraded.
  Need to get 2964 kB of archives.
  After this operation, 7056 kB of additional disk space will be used.
  Get:1 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 openssl amd64 3.0.2-0ubuntu1.7 [1183 kB]
  Get:2 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 ca-certificates all 20211016ubuntu0.22.04.1 [144 kB]
  Get:3 http://archive.ubuntu.com/ubuntu jammy/main amd64 libnghttp2-14 amd64 1.43.0-1build3 [76.3 kB]
  Get:4 http://archive.ubuntu.com/ubuntu jammy/main amd64 libpsl5 amd64 0.21.0-1.2build2 [58.4 kB]
  Get:5 http://archive.ubuntu.com/ubuntu jammy/main amd64 publicsuffix all 20211207.1025-1 [129 kB]
  Get:6 http://archive.ubuntu.com/ubuntu jammy/main amd64 libbrotli1 amd64 1.0.9-2build6 [315 kB]
  Get:7 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsasl2-modules-db amd64 2.1.27+dfsg2-3ubuntu1 [20.8 kB]
  Get:8 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsasl2-2 amd64 2.1.27+dfsg2-3ubuntu1 [53.9 kB]
  Get:9 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libldap-2.5-0 amd64 2.5.13+dfsg-0ubuntu0.22.04.1 [183 kB]
  Get:10 http://archive.ubuntu.com/ubuntu jammy/main amd64 librtmp1 amd64 2.4+20151223.gitfa8646d.1-2build4 [58.2 kB]
  Get:11 http://archive.ubuntu.com/ubuntu jammy/main amd64 libssh-4 amd64 0.9.6-2build1 [184 kB]
  Get:12 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcurl4 amd64 7.81.0-1ubuntu1.6 [290 kB]
  Get:13 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 curl amd64 7.81.0-1ubuntu1.6 [194 kB]
  Get:14 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libldap-common all 2.5.13+dfsg-0ubuntu0.22.04.1 [15.9 kB]
  Get:15 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsasl2-modules amd64 2.1.27+dfsg2-3ubuntu1 [57.5 kB]
  debconf: delaying package configuration, since apt-utils is not installed
  Fetched 2964 kB in 0s (7060 kB/s)
  Selecting previously unselected package openssl.
  (Reading database ... 4395 files and directories currently installed.)
  Preparing to unpack .../00-openssl_3.0.2-0ubuntu1.7_amd64.deb ...
  Unpacking openssl (3.0.2-0ubuntu1.7) ...
  Selecting previously unselected package ca-certificates.
  Preparing to unpack .../01-ca-certificates_20211016ubuntu0.22.04.1_all.deb ...
  Unpacking ca-certificates (20211016ubuntu0.22.04.1) ...
  Selecting previously unselected package libnghttp2-14:amd64.
  Preparing to unpack .../02-libnghttp2-14_1.43.0-1build3_amd64.deb ...
  Unpacking libnghttp2-14:amd64 (1.43.0-1build3) ...
  Selecting previously unselected package libpsl5:amd64.
  Preparing to unpack .../03-libpsl5_0.21.0-1.2build2_amd64.deb ...
  Unpacking libpsl5:amd64 (0.21.0-1.2build2) ...
  Selecting previously unselected package publicsuffix.
  Preparing to unpack .../04-publicsuffix_20211207.1025-1_all.deb ...
  Unpacking publicsuffix (20211207.1025-1) ...
  Selecting previously unselected package libbrotli1:amd64.
  Preparing to unpack .../05-libbrotli1_1.0.9-2build6_amd64.deb ...
  Unpacking libbrotli1:amd64 (1.0.9-2build6) ...
  Selecting previously unselected package libsasl2-modules-db:amd64.
  Preparing to unpack .../06-libsasl2-modules-db_2.1.27+dfsg2-3ubuntu1_amd64.deb ...
  Unpacking libsasl2-modules-db:amd64 (2.1.27+dfsg2-3ubuntu1) ...
  Selecting previously unselected package libsasl2-2:amd64.
  Preparing to unpack .../07-libsasl2-2_2.1.27+dfsg2-3ubuntu1_amd64.deb ...
  Unpacking libsasl2-2:amd64 (2.1.27+dfsg2-3ubuntu1) ...
  Selecting previously unselected package libldap-2.5-0:amd64.
  Preparing to unpack .../08-libldap-2.5-0_2.5.13+dfsg-0ubuntu0.22.04.1_amd64.deb ...
  Unpacking libldap-2.5-0:amd64 (2.5.13+dfsg-0ubuntu0.22.04.1) ...
  Selecting previously unselected package librtmp1:amd64.
  Preparing to unpack .../09-librtmp1_2.4+20151223.gitfa8646d.1-2build4_amd64.deb ...
  Unpacking librtmp1:amd64 (2.4+20151223.gitfa8646d.1-2build4) ...
  Selecting previously unselected package libssh-4:amd64.
  Preparing to unpack .../10-libssh-4_0.9.6-2build1_amd64.deb ...
  Unpacking libssh-4:amd64 (0.9.6-2build1) ...
  Selecting previously unselected package libcurl4:amd64.
  Preparing to unpack .../11-libcurl4_7.81.0-1ubuntu1.6_amd64.deb ...
  Unpacking libcurl4:amd64 (7.81.0-1ubuntu1.6) ...
  Selecting previously unselected package curl.
  Preparing to unpack .../12-curl_7.81.0-1ubuntu1.6_amd64.deb ...
  Unpacking curl (7.81.0-1ubuntu1.6) ...
  Selecting previously unselected package libldap-common.
  Preparing to unpack .../13-libldap-common_2.5.13+dfsg-0ubuntu0.22.04.1_all.deb ...
  Unpacking libldap-common (2.5.13+dfsg-0ubuntu0.22.04.1) ...
  Selecting previously unselected package libsasl2-modules:amd64.
  Preparing to unpack .../14-libsasl2-modules_2.1.27+dfsg2-3ubuntu1_amd64.deb ...
  Unpacking libsasl2-modules:amd64 (2.1.27+dfsg2-3ubuntu1) ...
  Setting up libpsl5:amd64 (0.21.0-1.2build2) ...
  Setting up libbrotli1:amd64 (1.0.9-2build6) ...
  Setting up libsasl2-modules:amd64 (2.1.27+dfsg2-3ubuntu1) ...
  Setting up libnghttp2-14:amd64 (1.43.0-1build3) ...
  Setting up libldap-common (2.5.13+dfsg-0ubuntu0.22.04.1) ...
  Setting up libsasl2-modules-db:amd64 (2.1.27+dfsg2-3ubuntu1) ...
  Setting up librtmp1:amd64 (2.4+20151223.gitfa8646d.1-2build4) ...
  Setting up libsasl2-2:amd64 (2.1.27+dfsg2-3ubuntu1) ...
  Setting up libssh-4:amd64 (0.9.6-2build1) ...
  Setting up openssl (3.0.2-0ubuntu1.7) ...
  Setting up publicsuffix (20211207.1025-1) ...
  Setting up libldap-2.5-0:amd64 (2.5.13+dfsg-0ubuntu0.22.04.1) ...
  Setting up ca-certificates (20211016ubuntu0.22.04.1) ...
  debconf: unable to initialize frontend: Dialog
  debconf: (TERM is not set, so the dialog frontend is not usable.)
  debconf: falling back to frontend: Readline
  debconf: unable to initialize frontend: Readline
  debconf: (Can't locate Term/ReadLine.pm in @INC (you may need to install the Term::ReadLine module) (@INC contains: /etc/perl /usr/local/lib/x86_64-linux-gnu/perl/5.34.0 /usr/local/share/perl/5.34.0 /usr/lib/x86_64-linux-gnu/perl5/5.34 /usr/share/perl5 /usr/lib/x86_64-linux-gnu/perl-base /usr/lib/x86_64-linux-gnu/perl/5.34 /usr/share/perl/5.34 /usr/local/lib/site_perl) at /usr/share/perl5/Debconf/FrontEnd/Readline.pm line 7.)
  debconf: falling back to frontend: Teletype
  Updating certificates in /etc/ssl/certs...
  124 added, 0 removed; done.
  Setting up libcurl4:amd64 (7.81.0-1ubuntu1.6) ...
  Setting up curl (7.81.0-1ubuntu1.6) ...
  Processing triggers for libc-bin (2.35-0ubuntu3.1) ...
  Processing triggers for ca-certificates (20211016ubuntu0.22.04.1) ...
  Updating certificates in /etc/ssl/certs...
  0 added, 0 removed; done.
  Running hooks in /etc/ca-certificates/update.d...
  done.
  COMMIT localhost:32000/curl:latest
  --> 7db5121257b
  Successfully tagged localhost:32000/curl:latest
  7db5121257b58f10241efd3eb155fe2aa9cfb7f0b9f875cd59d5e3b4a36efa21
  ````
  </details>
- Next, we'll login to our registry (`username` and `password` are both blank, just press _enter_)
  ````shell
  vagrant@worker1:~$ podman login localhost:32000 --tls-verify=false
  Username:
  Password:
  Login Succeeded!
  ````
- To conclude, we'll push the image to our private registry
  ````shell
  vagrant@worker1:~$ podman push localhost:32000/curl:latest --tls-verify=false
  Getting image source signatures
  Copying blob 554964fe94a0 done
  Copying blob 6515074984c6 skipped: already exists
  Copying config 7db5121257 done
  Writing manifest to image destination
  Storing signatures
  ````

🎉 Success! Image is now in our registry! Let's use it:

````shell
vagrant@worker1:~$ kubectl run --rm -it --image localhost:32000/curl:latest curl -- curl -I example.com
If you don't see a command prompt, try pressing enter.
warning: couldn't attach to pod/curl, falling back to streaming logs: unable to upgrade connection: container curl not found in pod curl_default
HTTP/1.1 200 OK
Content-Encoding: gzip
Accept-Ranges: bytes
Age: 554326
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Fri, 30 Dec 2022 16:50:08 GMT
Etag: "3147526947+gzip"
Expires: Fri, 06 Jan 2023 16:50:08 GMT
Last-Modified: Thu, 17 Oct 2019 07:18:26 GMT
Server: ECS (dcb/7F39)
X-Cache: HIT
Content-Length: 648

pod "curl" deleted
````

⎈ Working like a charm! Let's move on to more interesting use cases.

# Step 3 - Release our Application Container Image

With our container registry up and running, we'll now work on our application, a wiki written in `Go`. It's a simple
web application, from the Go tutorial, but it's perfect for us to use in this lab. Check out the 
[tutorial page](https://go.dev/doc/articles/wiki/), if you're interested. 

Before we set out to build and push images, however, our local `podman` must be able to pull container images itself
from the popular registry docker.io. To do so, run the following:
````shell
vagrant@worker1:~/lab1$ cat <<EOF | sudo tee -a /etc/containers/registries.conf.d/shortnames.conf
  # golang
  "golang" = "docker.io/library/golang"
EOF
  # golang
  "golang" = "docker.io/library/golang"
vagrant@worker1:~/lab1$ tail -n 5 /etc/containers/registries.conf.d/shortnames.conf
  "python" = "docker.io/library/python"
  # node
  "node" = "docker.io/library/node"
  # golang
  "golang" = "docker.io/library/golang"
````
- Great, we can now refer to "golang" when creating container images, and `podman` will fetch those from Docker hub.

Create a directory to store our application's files:
````shell
vagrant@worker1:~$ mkdir lab1 && cd lab1
vagrant@worker1:~/lab1$
````

Next, let's download the source code:
````shell
vagrant@worker1:~/lab1$ mkdir gowiki
vagrant@worker1:~/lab1$ curl -o gowiki/wiki.go "https://go.dev/doc/articles/wiki/final.go?m=text"
````

And create the View and Edit pages:
````shell
vagrant@worker1:~/lab1$ cat <<EOF > gowiki/edit.html
<h1>Editing {{.Title}}</h1>

<form action="/save/{{.Title}}" method="POST">
<div><textarea name="body" rows="20" cols="80">{{printf "%s" .Body}}</textarea></div>
<div><input type="submit" value="Save"></div>
</form>
EOF

vagrant@worker1:~/lab1$ cat <<EOF > gowiki/view.html
<h1>{{.Title}}</h1>

<p>[<a href="/edit/{{.Title}}">edit</a>]</p>

<div>{{printf "%s" .Body}}</div>
EOF
````

And finally, create the `Containerimage` file for our application:
````shell
vagrant@worker1:~/lab1$ cat <<EOF > Containerfile
FROM golang:1.19.4

WORKDIR /app

COPY gowiki .

RUN go mod init gowiki/wiki && \
    go build wiki.go
EOF
````

Everything's in place. Let's take a look at what we've got so far:
````shell
vagrant@worker1:~/lab1$ ls -la gowiki/
total 24
drwxrwxr-x 2 vagrant vagrant 4096 Jan 13 13:34 .
drwxrwxr-x 3 vagrant vagrant 4096 Jan  8 11:05 ..
-rw-rw-r-- 1 vagrant vagrant  216 Jan  8 11:11 edit.html
-rw-rw-r-- 1 vagrant vagrant   12 Jan  8 10:38 test.txt
-rw-rw-r-- 1 vagrant vagrant  100 Jan 13 13:34 view.html
-rw-rw-r-- 1 vagrant vagrant 2167 Jan  8 10:52 wiki.go
````

Next step, build our image:
````shell
vagrant@worker1:~/lab1$ podman build -t localhost:32000/gowiki:latest .
STEP 1/5: FROM golang:1.19.4
Resolved "golang" as an alias (/etc/containers/registries.conf.d/shortnames.conf)
Trying to pull docker.io/library/golang:1.19.4...
Getting image source signatures
Copying blob fa1d4c8d85a4 done
Copying blob 81283a9569ad done
Copying blob c796299bbbdd [======================================] 10.4MiB / 10.4MiB
Copying blob 32de3c850997 done
Copying blob c768848b86a2 done
Copying blob 3a3324c1d283 done
Copying blob ea43a17566db done
Copying config 6650307efe done
Writing manifest to image destination
Storing signatures
STEP 2/5: WORKDIR /app
--> cf28cce47b6
STEP 3/5: COPY gowiki .
--> 6fbc369f7b4
STEP 4/5: RUN go mod init gowiki/wiki
go: creating new go.mod: module gowiki/wiki
go: to add module requirements and sums:
	go mod tidy
--> 259cfd0e18e
STEP 5/5: RUN go build wiki.go
COMMIT localhost:32000/gowiki:latest
--> c29653c833a
Successfully tagged localhost:32000/gowiki:latest
c29653c833a1e6188778bb29183e8ccf82523fb2c9bc0d805267fad08bd26d1a
````

Let's try our application locally, before we publish our container image to the registry, and set out to create the
Deployment.

To run the application, we'll do so in _detached_ mode, meaning the container will be running in the background, so we
can still use out terminal session to interact with it. We'll also need to expose and map the container port, to send
HTTP requests to.
- We don't use `EXPOSE` in Containerimage because this image is meant to be used with Kubernetes, which will handle the
network for us. But since we need this mapping to test locally, we do it at runtime.

Here's what you'll run to launch the application:
````shell
vagrant@worker1:~/lab1$ podman run --name gowiki --rm --detach --expose=8080 -p 30080:8080 localhost:32000/gowiki:latest ./wiki
326f5b136575de00e071a824a3f7e44e23578d884904f2adabe25589c6217fe5
vagrant@worker1:~/lab1$ podman ps -a
CONTAINER ID  IMAGE                          COMMAND     CREATED       STATUS           PORTS                    NAMES
326f5b136575  localhost:32000/gowiki:latest  ./wiki      1 second ago  Up 1 second ago  0.0.0.0:35213->8080/tcp  gowiki
````
- Notice that, on the local host (_worker1_ instance), there is a mapping from `0.0.0.0:35213->container:8080`. This is
the port we'll use to connect to the application.

Now, let's make a request using `curl`, from _worker1_ directly:
````shell
vagrant@worker1:~/lab1$ curl localhost:35213/view/test
<h1>test</h1>

<p>[<a href="/edit/test">edit</a>]</p>

<div>Hello World
</div>
````

🎉 It's working! We have our application running, and we can interact with it. If you want to access it from your local
browser, check the instructions below:

<details>
  <summary><i>Exposing guest ports to host</i></summary>
  
  <a id="pf"> </a>
  
  If you want access a specific port inside the VM, from your laptop, you'll need to first configure a port-forward in
  the Vagrant configuration. Simply change the configuration for one or more VMs, as follows:
  ````text
  config.vm.define :worker1 do |worker1|
    worker1.vm.box = "ubuntu/jammy64"
    worker1.vm.hostname = "worker1"
    worker1.vm.network "private_network", ip: "192.168.56.11", virtualbox__intnet: "labnet"
    worker1.vm.network "forwarded_port", guest: 30080, host: 30080
  end
  ````
  - You'll need to modify the VM after this change.
    - Run `vagrant reload`
  - This will map port `30080` in your local machine, to port `30080` in the VM.
    - Be careful choosing a port, so not no overlap with an existing one. Use `lsof -iTCP -sTCP:LISTEN -n -P` to find
      which ports are in LISTEN.
  - When selecting a port to `EXPOSE` in the container, map it to `30080`. Change the `podman run` command to now map
    a local port to the container port:
    ````shell
    vagrant@worker1:~$ podman run --name gowiki --rm --detach --expose=8080 -p 30080:8080 localhost:32000/gowiki:latest ./wiki
    e99356630fd0858bc3a63a4614b7c904d10fd87263e8c3f908659fe21e782ff5
    vagrant@worker1:~$ podman ps -a
    CONTAINER ID  IMAGE                          COMMAND     CREATED        STATUS            PORTS                    NAMES
    e99356630fd0  localhost:32000/gowiki:latest  ./wiki      3 seconds ago  Up 3 seconds ago  0.0.0.0:30080->8080/tcp  gowiki
    vagrant@worker1:~$
    ````
</details>

Now that we've tested our application, let's publish it in our container registry:
````shell
vagrant@worker1:~$ podman push localhost:32000/gowiki:latest --tls-verify=false
Getting image source signatures
Copying blob 7354e83da007 done
Copying blob c284f546974c done
Copying blob 8098fd9e488d done
Copying blob 148bea68f01d done
Copying blob fc53c9ff7735 done
Copying blob 38c819c72899 done
Copying blob f77a030cb1dd done
Copying blob bb2453e12947 done
Copying blob 4efcd4003c84 done
Copying blob aa278d9b9e12 done
Copying config c29653c833 done
Writing manifest to image destination
Storing signatures
````

✅ Done! On to the Deployment!

# Step 4 - Create the Application Deployment

We're going to create a simple Deployment for our simple application. There's nothing too fancy, and by now, all of what
we're going to write in our manifest, should look familiar.

Write the following manifest to `deployment.yaml`:
````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: gowiki
  name: gowiki
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
          image: localhost:32000/gowiki:latest
          ports:
            - containerPort: 8080
              name: http  
````

Before we set out to apply our manifest, we need to create our application's namespace:
````shell
vagrant@worker1:~$ kubectl create namespace gowiki
namespace/gowiki created
vagrant@worker1:~$ kubectl get namespaces gowiki
NAME     STATUS   AGE
gowiki   Active   10s
````

Ok, now we can apply:
````shell
vagrant@worker1:~/lab1$ kubectl apply -f deployment.yaml
deployment.apps/gowiki created
````

Let's see if it's running nicely:
````shell
vagrant@worker1:~/lab1$ kubectl get deployments.apps --namespace gowiki
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
gowiki   1/1     1            1           109s
vagrant@worker1:~/lab1$ kubectl describe deployments.apps --namespace gowiki
Name:                   gowiki
Namespace:              gowiki
CreationTimestamp:      Fri, 13 Jan 2023 15:08:03 +0000
Labels:                 app=gowiki
                        type=web
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=gowiki
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=gowiki
           type=web
  Containers:
   gowiki:
    Image:      localhost:32000/gowiki:latest
    Port:       8080/TCP
    Host Port:  0/TCP
    Command:
      ./wiki
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   gowiki-d6f5bb5dc (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  119s  deployment-controller  Scaled up replica set gowiki-d6f5bb5dc to 1
````

🎉 Success! Our application is deployed! To interact with the application, and see if it's working as intended, we'll
make use of `kubectl port-forward`, a feature that allows us to access the container port from outside the cluster:
````shell
vagrant@worker1:~/lab1$ kubectl port-forward pod/gowiki-d6f5bb5dc-t5wxs --namespace gowiki 30080:8080 &
[1] 60073 
Forwarding from 127.0.0.1:30080 -> 8080
Forwarding from [::1]:30080 -> 8080
````
- We'll send it to the background, with `&`. Press \<enter> to return to your terminal. 
- If you get an error, you probably still have the container, from earlier testing, still running. Kill it, and retry: 
  `````shell
  vagrant@worker1:~/lab1$ kubectl port-forward pod/gowiki-d6f5bb5dc-t5wxs --namespace gowiki 30080:8080
  Unable to listen on port 30080: Listeners failed to create with the following errors: [unable to create listener: Error listen tcp4 127.0.0.1:30080: bind: address already in use unable to create listener: Error listen tcp6 [::1]:30080: bind: address already in use]
  error: unable to listen on any of the requested ports: [{30080 8080}]
  vagrant@worker1:~/lab1$ podman container kill gowiki
  gowiki
  `````

Now, if we send a request to our application on port `30080`, from our local session on the VM, we should be able to get
a response:
````shell
vagrant@worker1:~/lab1$ curl localhost:30080/view/test
Handling connection for 30080
<h1>test</h1>

<p>[<a href="/edit/test">edit</a>]</p>

<div>Hello World
</div>
````
There it is! We've successfully accessed the application, running in our cluster, from the outside!
- To stop the port-forward, simply run `kill %1`:
  ````shell
  vagrant@worker1:~/lab1$ kill %1
  vagrant@worker1:~/lab1$ jobs
  [1]+  Terminated              microk8s kubectl port-forward pod/gowiki-d6f5bb5dc-t5wxs --namespace gowiki 30080:8080
  ````

<details>
   <summary><i>But what if you want to access it from your browser, outside the VM?</i></summary>
  
  This is a subject that goes beyond what we've covered in the _Defining Workloads_ tutorial series. This is related to
  network and service discovery, but if you want to access your application from your browser, do the following:
  - Configure the port-forward in the Vagrant file.
    - Check the [_Exposing guest ports to host_](#pf) section.
    - Make sure the `kubectl port-forward` is stopped.
  - Create a file named `service.yaml`:
    ````yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: gowiki
      namespace: gowiki
      labels:
        app: gowiki
    spec:
      ports:
        - name: http
          port: 8080
          targetPort: 8080
          nodePort: 30080
          protocol: TCP
      selector:
        app: gowiki
      type: NodePort
    ````
  - Apply the manifest to the cluster:
  ````shell
  vagrant@worker1:~/lab1$ kubectl apply -f service.yaml
  service/gowiki created
  ````
  - Access `localhost:30080` in your browser.
</details>

# Step 5 - Health Checks

Our application is deployed and running smoothly. While the Deployment is configured nicely, we're lacking the 
capability of knowing if our application is ready to serve requests, or if it's running at all! Granted, the container
lifecyle will kick in if the application suddenly crashes, but as a best-practice, we should always have liveness and
readiness probes.

For this case, we'll use liveness and readiness probes not checking our application endpoint, but simply checking if the
container is listening on our application port (`8080`).

Go back to the `deployment.yaml` file and add the `livenessProbe` and `readinessProbe` sections below:
````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: gowiki
  name: gowiki
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
          image: localhost:32000/gowiki:latest
          ports:
            - containerPort: 8080
              name: http 
          livenessProbe:
            tcpSocket:
              port: 8080
          readinessProbe:
            tcpSocket:
              port: 8080
````

<details>
  <summary>Apply the manifest, and check out the Pod description:</summary>

````shell
vagrant@worker1:~/lab1$ kubectl apply -f deployment.yaml
deployment.apps/gowiki configured
vagrant@worker1:~/lab1$ kubectl describe pod --namespace gowiki gowiki-757b4769c7-xh4xj
Name:             gowiki-757b4769c7-xh4xj
Namespace:        gowiki
Priority:         0
Service Account:  default
Node:             worker2/192.168.56.12
Start Time:       Fri, 13 Jan 2023 18:20:12 +0000
Labels:           app=gowiki
                  pod-template-hash=757b4769c7
                  type=web
Annotations:      cni.projectcalico.org/containerID: 33c6286ba6391f86b8f9d62d4f8ebc2f66034519fc5f992177b7bb8f47d68720
                  cni.projectcalico.org/podIP: 10.1.189.84/32
                  cni.projectcalico.org/podIPs: 10.1.189.84/32
Status:           Running
IP:               10.1.189.84
IPs:
  IP:           10.1.189.84
Controlled By:  ReplicaSet/gowiki-757b4769c7
Containers:
  gowiki:
    Container ID:  containerd://363c95296f16d8ae90fd7e66c98e206333bdb414e0ceacca5f6b0c4c96fb8808
    Image:         localhost:32000/gowiki:latest
    Image ID:      localhost:32000/gowiki@sha256:173d044c00e83f2a7ac91906be589a546624f25ef7a88262024effbac500e222
    Port:          8080/TCP
    Host Port:     0/TCP
    Command:
      ./wiki
    State:          Running
      Started:      Fri, 13 Jan 2023 18:20:13 +0000
    Ready:          True
    Restart Count:  0
    Liveness:       tcp-socket :8080 delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:      tcp-socket :8080 delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-bsth7 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-bsth7:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  48s   default-scheduler  Successfully assigned gowiki/gowiki-757b4769c7-xh4xj to worker2
  Normal  Pulling    50s   kubelet            Pulling image "localhost:32000/gowiki:latest"
  Normal  Pulled     50s   kubelet            Successfully pulled image "localhost:32000/gowiki:latest" in 33.498614ms
  Normal  Created    50s   kubelet            Created container gowiki
  Normal  Started    50s   kubelet            Started container gowiki
````
</details>

You'll see that the Pod now has liveness and readiness probes:
````text
Liveness:       tcp-socket :8080 delay=0s timeout=1s period=10s #success=1 #failure=3
Readiness:      tcp-socket :8080 delay=0s timeout=1s period=10s #success=1 #failure=3
````

This is interesting, but it didn't change anything in the application. Change the `readinessProbe` port to something
else, say `8081`, and see what happens.
````shell
vagrant@worker1:~/lab1$ kubectl apply -f deployment.yaml
deployment.apps/gowiki configured
vagrant@worker1:~/lab1$ kubectl get pods --namespace gowiki
NAME                      READY   STATUS    RESTARTS   AGE
gowiki-757b4769c7-xh4xj   1/1     Running   0          6m3s
gowiki-65f7c49c56-pwhww   0/1     Running   0          23s
````

Now our new Deployment rollout is stuck:
  ````shell
  vagrant@worker1:~/lab1$ kubectl rollout status deployment/gowiki --namespace gowiki
  Waiting for deployment "gowiki" rollout to finish: 1 old replicas are pending termination...
  ````

This is because the Pod failed the readiness check. If we `kubect describe` the Pod, we can see why:
````shell
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  10s               default-scheduler  Successfully assigned gowiki/gowiki-65f7c49c56-pwhww to worker2
  Normal   Pulling    12s               kubelet            Pulling image "localhost:32000/gowiki:latest"
  Normal   Pulled     12s               kubelet            Successfully pulled image "localhost:32000/gowiki:latest" in 28.263919ms
  Normal   Created    12s               kubelet            Created container gowiki
  Normal   Started    12s               kubelet            Started container gowiki
  Warning  Unhealthy  3s (x4 over 11s)  kubelet            Readiness probe failed: dial tcp 10.1.189.85:8081: connect: connection refused
````
- Makes sense, since there's nothing listening on that port.


Rolling back to the previous Deployment will fix the issue: 
````shell
vagrant@worker1:~/lab1$ kubectl rollout undo --namespace gowiki deployment gowiki
deployment.apps/gowiki rolled back
vagrant@worker1:~/lab1$ kubectl rollout status deployment/gowiki --namespace gowiki
deployment "gowiki" successfully rolled out
vagrant@worker1:~/lab1$ kubectl get deployment --namespace gowiki
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
gowiki   1/1     1            1           3h27m
vagrant@worker1:~/lab1$ kubectl get pods --namespace gowiki
NAME                      READY   STATUS    RESTARTS   AGE
gowiki-757b4769c7-xh4xj   1/1     Running   0          16m
````
- By rolling back, we only reverted back to the manifest we had earlier. The latest, the one with the wrong port is 
  still there, and we can `rollout undo` into it, if we specify `--to-revision 3`:
  ````shell
  vagrant@worker1:~/lab1$ kubectl rollout undo --namespace gowiki deployment gowiki --to-revision 3
  deployment.apps/gowiki rolled back
  vagrant@worker1:~/lab1$ kubectl get pods --namespace gowiki
  NAME                      READY   STATUS    RESTARTS   AGE
  gowiki-757b4769c7-xh4xj   1/1     Running   0          17m
  gowiki-65f7c49c56-n2qrl   0/1     Running   0          0s
  vagrant@worker1:~/lab1$ kubectl rollout history --namespace gowiki deployment gowiki
  deployment.apps/gowiki
  REVISION  CHANGE-CAUSE
  1         <none>
  4         <none>
  5         <none>
  ````
  - However, with this, we now have revision 4, which is the old revision 2 (the correct one), and created a revision 5,
    which is the one we're currently at, with the wrong port.
  - Simply re-run the `rollback undo` to return to the sane Deployment.

And we're done! We've deployed our own application, and it even has health checks! 💪

If you want to continue experimenting with this Deployment, and use some of what we learned in _Defining Workloads_, 
here are some suggestions for challenging yourself:
- We've seen that the application reads `.html` files from its root directory, to make wiki pages available right from 
  the start. Without modifying the container image, try adding more files to the app.
  - Hints
      <details>
      <summary></summary>
    
      Create a configMap with the file, and mount it in the container, using `subPath`
      </details>
      <details>
      <summary></summary>
    
      Add an `initContainer` that pulls the HTML file from the internet. You'll need to create a `volume`, mount it 
      both containers, for this to work. Use an `emptyDir` volume type for this.
      </details>
