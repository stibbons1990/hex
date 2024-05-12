---
title:  "Single-node Kubernetes cluster on Ubuntu Studio desktop (rapture)"
date:   2024-05-12 15:05:12 +0200
categories: ubuntu studio desktop linux kubernetes docker
---

For a bit more than a year, I've been running self-hosted
services on a
[single-node Kubernetes cluster on Ubuntu server]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html)
and, while it has presented some troubles to shoot, I have
grown used to the advantages of deploying services far more
easily, without having to worry too much about their
dependencies, getting automatic updates, and even making many
of them available over HTTPS with good SSL certificates.
Now I want to *enjoy* some of those advantages in my desktop
PC: **Rapture**.

{% assign media = site.baseurl | append: "/assets/media/" | append:  page.path | replace: ".md","" | replace: "_posts/",""  %}

## Motivation

In the last couple of months, a couple of *incidents* have left
me thinking it may be time to upgrade the cluster.

First, following the removal of
[Legacy Package Repositories](https://kubernetes.io/blog/2023/08/31/legacy-package-repository-deprecation/),
I had to
[update Kubernetes to the Community-Owned Package Repositories]({{ site.baseurl }}/2024/03/05/updating-kubernetes-to-the-community-owned-package-repositories.html),
which didn't *directly* break anything but there *was*
[an unexpected breakage]({{ site.baseurl }}/2024/03/05/updating-kubernetes-to-the-community-owned-package-repositories.html#an-unexpected-breakage)
due to a
[minor version mismatch](https://kubernetes.io/releases/version-skew-policy/#supported-versions),
after which the cluster was left at version **1.26** as this
was the one originall installed for the
[single-node Kubernetes cluster on]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html).

Then, just a few weeks later, the
[Kubernetes Certificate Expired]({{ site.baseurl }}/2024/03/22/kubectl-certificate-expired.html)
and eventually the solution was to perform a cumbersome
[manual certificate renewal]({{ site.baseurl }}/2024/03/22/kubectl-certificate-expired.html#manual-certificate-renewal).
While I'm still not entirely convince that it *will* work,
apparently certificates *should* be automatically renewed when
the Kubernetes cluster is upgraded to a higher *minor version*.

These left me with the nagging feeling that Kubernetes clusters
need to be periodically upgraded, but the stark impression that
*one does not simply upgrade*, even though at least 
[Philpep's Kubernetes the self-hosted single node way](https://philpep.org/blog/kubernetes-the-self-hosted-way/)
claims *it wasn't so bad*:

> Handling upgrade with kubeadm is quite simple as long as you
> read the upgrade notes carefully. For the latest releases
> I just ran:
> ```
> $ export v=1.14.1
> $ apt-get update
> $ apt-get install kubeadm=$v-00 kubelet=$v-00
> $ kubeadm upgrade plan
> [...]
> $ kubeadm upgrade apply v$v
> ```

At any rate, it would be better to have a second single-node
cluster that is not running *critical* services, which is only
to say *services I would be rather inconvenienced without*, if
only to try things out with the peace of mind of not riksing
*breaking production the cluster*.

### More Motivation

On the other hand, other incidents went rather well, e.g. when
[Getbukkit Expired]({{ site.baseurl }}/2024/03/15/getbukkit-expired.html)
the solutions where relativly easy to apply and I suspect
it was easier to update a single YAML file and `kubectl apply`
than applying the same changes on a bespoke environment would
have been.

I have also been wanting to
[migrate Plex Media Server to Kubernetes]({{ site.baseurl }}/2023/09/16/migrating-a-plex-media-server-to-kubernetes.html)
on my desktop PC, even though I barely use it.
Precisely because I barely use this one, it would be nice to
deploy it in Kubernetes and forget about it, letting Kubernetes
handle updates which I'm not bothering with (not using it).

I might also want to make more risky experiments with a
[Minecraft Java Server for Bedrock clients]({{ site.baseurl }}/2023/08/10/running-minecraft-java-server-for-bedrock-clients-on-kubernetes.html),
something I'd feel better doing on my PC with a `world` nobody
care about.

But what I think would be more interesting is trying some other
applications I have not been able to run in Kubernetes so far, mostly because they are much more CPU-hungry than what a low-end
Intel NUC can handle:

*  [Audiobookshelf]({{ site.baseurl }}/2024/02/28/audiobookshelf-on-kubernetes.html) is *surprisingly* CPU
   intensive, not sure why but it is the one service that makes
   the CPU run hotter on a regular basis, although always in
   short spikes and never much above 100% (a single CPU core).
*  [GitLab Helm chart](https://docs.gitlab.com/charts/)
   requires enough CPU to be allocated that, together with
   just the basic single-node cluster allocations, it exceeded
   the limit of the available 4 CPU cores. Besides, running
   [CI/CD pipelines](https://docs.gitlab.com/ee/ci/pipelines/)
   would likely take too long on a system with few CPU cores.
*  [PhotoPrism](https://docs.photoprism.app/getting-started/advanced/kubernetes/)
   is quite CPU hungry too, as is only to be expected given the
   features it offers, so it would only run well on a high-end
   CPU and that's one I only have on my desktop PC. Besides,
   that PC is the one where the files (photos and videos) live.

## GitHub Repository

From this point on, adding deployments to the cluster will
be mostly based on YAML files and these are best stored in
a revision controlled environment. A GitHub repository
(e.g. `kubernetes-deployments`) will do just fine, and with
the idea of later upgrading the cluster to newer versions of
Kubernetes, it may be a good idea to keep deployments in
separate directories for each Kubernetes version:

```
$ git clone git@github.com:xxxx/kubernetes-deployments.git
$ cd kubernetes-deployments/
$ mkdir 1.26
$ cd 1.26
```

## Install Kubernetes

[Kubernetes installation in Lexicon]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html#kubernetes)
was based on 
[Install Kubernetes Cluster on Ubuntu 22.04 using kubeadm](https://computingforgeeks.com/install-kubernetes-cluster-ubuntu-jammy/),
[How to install Kubernetes on Ubuntu 22.04 Jammy Jellyfish Linux](https://linuxconfig.org/how-to-install-kubernetes-on-ubuntu-22-04-jammy-jellyfish-linux)
and
[How to Install Kubernetes Cluster on Ubuntu 22.04](https://www.linuxtechi.com/install-kubernetes-on-ubuntu-22-04/).
These article were all very helpful and I would highly recommend
following them, or their updated versions whenever available,
to anyone getting started with Kubernetes for the first time.

Being now somewhat more familiar with Kubernetes, this time
I will be following the Kubernetes Documentation at
[kubernetes.io/docs](https://kubernetes.io/docs) to learn more
details *first-hand* from the official documentation.

To install the older **1.26** version, which is the one running
in **Lexicon**, so that the future upgrade/s to newer versions
can be tested in **Rapture**, the first step is to jump from the
[Available Documentation Versions](https://kubernetes.io/docs/home/supported-doc-versions/)
page to the [1.26 documentation](https://v1-26.docs.kubernetes.io/docs/home/supported-doc-versions/)
and from there to the starting point:
[Getting started](https://v1-26.docs.kubernetes.io/docs/setup/).

Right from the start, things get *confusing*:
[Download Kubernetes](https://v1-26.docs.kubernetes.io/releases/download/) lists a number of *Container Images* and points to
[downloadkubernetes.com](https://www.downloadkubernetes.com/)
to find the binaries, but there is no clear indication as to
*which* binaries are *actually required*, or recommended.

There is not even a mention of the possiblity to install these
binaries through more convenient means such as the
*native package management* which is explained only later,
in the documentation to
[Install and Set Up kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux),
in particular
[using native package management](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)
for *Debian-based distributions*:

```
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.26/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.26/deb/ /' \
  | tee /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.26/deb/ /

# apt-get update
...
Get:4 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  InRelease [1,192 B]
Get:11 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  Packages [22.1 kB]

# apt-get install -y kubectl
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  kubectl
0 upgraded, 1 newly installed, 0 to remove and 13 not upgraded.
Need to get 10.2 MB of archives.
After this operation, 48.2 MB of additional disk space will be used.
Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  kubectl 1.26.15-1.1 [10.2 MB]
Fetched 10.2 MB in 1s (16.1 MB/s)
Selecting previously unselected package kubectl.
(Reading database ... 636365 files and directories currently installed.)
Preparing to unpack .../kubectl_1.26.15-1.1_amd64.deb ...
Unpacking kubectl (1.26.15-1.1) ...
Setting up kubectl (1.26.15-1.1) ...
```

Additional commands may be necessary in a brand-new Ubuntu
desktop install, but in my case `/etc/apt/keyrings` akready
existed and `apt-transport-https ca-certificates curl` were
all already installed.

With only `kubectl` installed, pretty much anything that one
can try will (inevitably) fail because there is no cluster to
interact with:

```
# kubectl version --output=yaml
clientVersion:
  buildDate: "2024-03-14T01:05:39Z"
  compiler: gc
  gitCommit: 1649f592f1909b97aa3c2a0a8f968a3fd05a7b8b
  gitTreeState: clean
  gitVersion: v1.26.15
  goVersion: go1.21.8
  major: "1"
  minor: "26"
  platform: linux/amd64
kustomizeVersion: v4.5.7

The connection to the server localhost:8080 was refused - did you specify the right host or port?

# kubectl cluster-info
E0510 20:41:21.723009 2656643 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0510 20:41:21.723354 2656643 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0510 20:41:21.724591 2656643 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0510 20:41:21.724761 2656643 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0510 20:41:21.726089 2656643 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
The connection to the server localhost:8080 was refused - did you specify the right host or port?

# kubectl cluster-info dump
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

In addition to `kubectl` at least the following 2 packages are
required to start and run a Kubernetes cluster:

*  `kubeadm`: Command-line utility for administering a Kubernetes cluster.
*  `kubectl`: Command-line utility for interacting with a Kubernetes cluster.
*  `kubelet`: Node agent for Kubernetes clusters.

Although far from being clearly explained, or even mentioned,
[anywhere under docs.kubernetes.io](https://www.google.com/search?q=site%3Adocs.kubernetes.io+kubelet+kubeadm), these
components can be installed as simply `kubectl`:

```
# apt install -y kubelet kubeadm kubectl
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
kubectl is already the newest version (1.26.15-1.1).
The following additional packages will be installed:
  conntrack cri-tools ebtables ethtool kubernetes-cni
The following NEW packages will be installed:
  conntrack cri-tools ebtables ethtool kubeadm kubelet kubernetes-cni
0 upgraded, 7 newly installed, 0 to remove and 13 not upgraded.
Need to get 77.8 MB of archives.
After this operation, 296 MB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu jammy/main amd64 conntrack amd64 1:1.4.6-2build2 [33.5 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy/main amd64 ebtables amd64 2.0.11-4build2 [84.9 kB]
Get:4 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 ethtool amd64 1:5.16-1ubuntu0.1 [207 kB]
Get:3 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  cri-tools 1.26.0-1.1 [19.0 MB]
Get:5 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  kubernetes-cni 1.2.0-2.1 [27.6 MB]
Get:6 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  kubelet 1.26.15-1.1 [21.1 MB]
Get:7 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  kubeadm 1.26.15-1.1 [9,842 kB]
Fetched 77.8 MB in 1s (54.5 MB/s)  
Selecting previously unselected package conntrack.
(Reading database ... 636369 files and directories currently installed.)
Preparing to unpack .../0-conntrack_1%3a1.4.6-2build2_amd64.deb ...
Unpacking conntrack (1:1.4.6-2build2) ...
Selecting previously unselected package cri-tools.
Preparing to unpack .../1-cri-tools_1.26.0-1.1_amd64.deb ...
Unpacking cri-tools (1.26.0-1.1) ...
Selecting previously unselected package ebtables.
Preparing to unpack .../2-ebtables_2.0.11-4build2_amd64.deb ...
Unpacking ebtables (2.0.11-4build2) ...
Selecting previously unselected package ethtool.
Preparing to unpack .../3-ethtool_1%3a5.16-1ubuntu0.1_amd64.deb ...
Unpacking ethtool (1:5.16-1ubuntu0.1) ...
Selecting previously unselected package kubernetes-cni.
Preparing to unpack .../4-kubernetes-cni_1.2.0-2.1_amd64.deb ...
Unpacking kubernetes-cni (1.2.0-2.1) ...
Selecting previously unselected package kubelet.
Preparing to unpack .../5-kubelet_1.26.15-1.1_amd64.deb ...
Unpacking kubelet (1.26.15-1.1) ...
Selecting previously unselected package kubeadm.
Preparing to unpack .../6-kubeadm_1.26.15-1.1_amd64.deb ...
Unpacking kubeadm (1.26.15-1.1) ...
Setting up conntrack (1:1.4.6-2build2) ...
Setting up ebtables (2.0.11-4build2) ...
Setting up cri-tools (1.26.0-1.1) ...
Setting up kubernetes-cni (1.2.0-2.1) ...
Setting up ethtool (1:5.16-1ubuntu0.1) ...
Setting up kubelet (1.26.15-1.1) ...
Setting up kubeadm (1.26.15-1.1) ...
Processing triggers for man-db (2.10.2-1) ...
```

Among other tools, this installs other critical components to
run a Kubernetes cluster

*  `cri-tools`: Command-line utility for interacting with a
   container runtime.
*  `kubernetes-cni`: Binaries required to provision kubernetes
   container networking.

## Install container runtime

And now we come to the next major steps in the installation of
a new Kubernetes cluster:
*select a [container runtime](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/)*.

### Prerequisites

[Forwarding IPv4 and letting iptables see bridged traffic](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic)
are already enabled in Ubuntu:

```
# lsmod | egrep 'overlay|bridge'
bridge                311296  1 br_netfilter
stp                    16384  1 bridge
llc                    16384  2 bridge,stp
overlay               151552  1

# sysctl -a | egrep 'net.ipv4.ip_forward|net.bridge.bridge-nf-call-ip'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

### Install containerd

To install containerd, follow the instructions on 
[getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md),
in particular
[Option 2: From `apt-get`](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf)
which leads to the instructions to
[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/).

Normally this starts by uninstalling conflicting packages, that
is those provided by Ubuntu, before installing the official
version of Docker Engine.
In the case of Rapture, there are a few packages installed
from learning excercises over a year ago:

```
# for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do 
  dpkg -l $pkg 2>/dev/null | grep ^ii
done
ii  docker.io      24.0.5-0ubuntu1~22.04.1 amd64        Linux container runtime
ii  docker-compose 1.29.2-1                all          define and run multi-container Docker applications with YAML
ii  containerd     1.7.2-0ubuntu1~22.04.1  amd64        daemon to control runC
ii  runc           1.1.7-0ubuntu1~22.04.2  amd64        Open Container Project - runtime

# docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED        STATUS        PORTS      NAMES
e6a7446a39d5   mariadb:10.11   "docker-entrypoint.s…"   8 months ago   Up 26 hours   3306/tcp   photoprism_mariadb_1

# docker images
REPOSITORY              TAG       IMAGE ID       CREATED         SIZE
mariadb                 10.11     58df8de36e1c   8 months ago    403MB
photoprism/photoprism   latest    c7bf390f2ce9   9 months ago    1.84GB
website                 latest    e4ff4dff6768   14 months ago   57.5MB
<none>                  <none>    e000dde7140f   14 months ago   57.5MB
nginx                   alpine    2bc7edbc3cf2   15 months ago   40.7MB
```

These all begin very old leftovers, they are all to be
thoroughly removed:

```
# apt-get purge -y docker.io docker-compose containerd runc
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages were automatically installed and are no longer required:
  bridge-utils pigz python3-docker python3-dockerpty python3-dotenv python3-jsonschema
  python3-pyrsistent python3-texttable python3-websocket ubuntu-fan
Use 'apt autoremove' to remove them.
The following packages will be REMOVED:
  containerd* docker-compose* docker.io* runc*
0 upgraded, 0 newly installed, 4 to remove and 2 not upgraded.
After this operation, 266 MB disk space will be freed.
(Reading database ... 636455 files and directories currently installed.)
Removing docker.io (24.0.5-0ubuntu1~22.04.1) ...
'/usr/share/docker.io/contrib/nuke-graph-directory.sh' -> '/var/lib/docker/nuke-graph-directory.sh'
Warning: Stopping docker.service, but it can still be activated by:
  docker.socket
Removing containerd (1.7.2-0ubuntu1~22.04.1) ...
Removing docker-compose (1.29.2-1) ...
Removing runc (1.1.7-0ubuntu1~22.04.2) ...
Processing triggers for man-db (2.10.2-1) ...
(Reading database ... 636121 files and directories currently installed.)
Purging configuration files for docker.io (24.0.5-0ubuntu1~22.04.1) ...

Nuking /var/lib/docker ...
  (if this is wrong, press Ctrl+C NOW!)

+ sleep 10

+ rm -rf /var/lib/docker/buildkit /var/lib/docker/containers /var/lib/docker/engine-id /var/lib/docker/image /var/lib/docker/network /var/lib/docker/nuke-graph-directory.sh /var/lib/docker/overlay2 /var/lib/docker/plugins /var/lib/docker/runtimes /var/lib/docker/swarm /var/lib/docker/tmp /var/lib/docker/trust /var/lib/docker/volumes
dpkg: warning: while removing docker.io, directory '/etc/docker' not empty so not removed
Purging configuration files for containerd (1.7.2-0ubuntu1~22.04.1) ...

# rm -rf /var/lib/containerd /var/lib/docker/

# apt autoremove -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages will be REMOVED:
  bridge-utils pigz python3-docker python3-dockerpty python3-dotenv python3-jsonschema
  python3-pyrsistent python3-texttable python3-websocket ubuntu-fan
0 upgraded, 0 newly installed, 10 to remove and 2 not upgraded.
After this operation, 1,896 kB disk space will be freed.
(Reading database ... 636120 files and directories currently installed.)
Removing ubuntu-fan (0.12.16) ...
ubuntu-fan: removing default /etc/network/fan configuration
Removing bridge-utils (1.7-1ubuntu3) ...
Removing pigz (2.6-1) ...
Removing python3-docker (5.0.3-1) ...
Removing python3-dockerpty (0.4.1-2) ...
Removing python3-dotenv (0.19.2-1) ...
Removing python3-jsonschema (3.2.0-0ubuntu2) ...
Removing python3-pyrsistent:amd64 (0.18.1-1build1) ...
Removing python3-texttable (1.6.4-1) ...
Removing python3-websocket (1.2.3-1) ...
Processing triggers for man-db (2.10.2-1) ...
```

With all those old packages out of the we, we proceed to
set up and install Docker Engine from 
[Docker's apt repository](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).

```
# curl \
  -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

# echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu   jammy stable

# apt-get update
...
Get:10 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease [7,553 B]
Get:14 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease [7,449 B]

# apt-get install -y \
  docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  docker-ce-rootless-extras libslirp0 pigz slirp4netns
Suggested packages:
  aufs-tools cgroupfs-mount | cgroup-lite
The following NEW packages will be installed:
  containerd.io docker-buildx-plugin docker-ce docker-ce-cli docker-ce-rootless-extras
  docker-compose-plugin libslirp0 pigz slirp4netns
0 upgraded, 9 newly installed, 0 to remove and 2 not upgraded.
Need to get 121 MB of archives.
After this operation, 434 MB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu jammy/universe amd64 pigz amd64 2.6-1 [63.6 kB]
Get:2 https://download.docker.com/linux/ubuntu jammy/stable amd64 containerd.io amd64 1.6.31-1 [29.8 MB]
Get:3 http://archive.ubuntu.com/ubuntu jammy/main amd64 libslirp0 amd64 4.6.1-1build1 [61.5 kB]
Get:4 http://archive.ubuntu.com/ubuntu jammy/universe amd64 slirp4netns amd64 1.0.1-2 [28.2 kB]
Get:5 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-buildx-plugin amd64 0.14.0-1~ubuntu.22.04~jammy [29.7 MB]
Get:6 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-ce-cli amd64 5:26.1.2-1~ubuntu.22.04~jammy [14.6 MB]
Get:7 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-ce amd64 5:26.1.2-1~ubuntu.22.04~jammy [25.3 MB]
Get:8 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-ce-rootless-extras amd64 5:26.1.2-1~ubuntu.22.04~jammy [9,319 kB]
Get:9 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-compose-plugin amd64 2.27.0-1~ubuntu.22.04~jammy [12.5 MB]
Fetched 121 MB in 1s (95.8 MB/s)                
Selecting previously unselected package pigz.
(Reading database ... 635829 files and directories currently installed.)
Preparing to unpack .../0-pigz_2.6-1_amd64.deb ...
Unpacking pigz (2.6-1) ...
Selecting previously unselected package containerd.io.
Preparing to unpack .../1-containerd.io_1.6.31-1_amd64.deb ...
Unpacking containerd.io (1.6.31-1) ...
Selecting previously unselected package docker-buildx-plugin.
Preparing to unpack .../2-docker-buildx-plugin_0.14.0-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-buildx-plugin (0.14.0-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package docker-ce-cli.
Preparing to unpack .../3-docker-ce-cli_5%3a26.1.2-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-ce-cli (5:26.1.2-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package docker-ce.
Preparing to unpack .../4-docker-ce_5%3a26.1.2-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-ce (5:26.1.2-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package docker-ce-rootless-extras.
Preparing to unpack .../5-docker-ce-rootless-extras_5%3a26.1.2-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-ce-rootless-extras (5:26.1.2-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package docker-compose-plugin.
Preparing to unpack .../6-docker-compose-plugin_2.27.0-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-compose-plugin (2.27.0-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package libslirp0:amd64.
Preparing to unpack .../7-libslirp0_4.6.1-1build1_amd64.deb ...
Unpacking libslirp0:amd64 (4.6.1-1build1) ...
Selecting previously unselected package slirp4netns.
Preparing to unpack .../8-slirp4netns_1.0.1-2_amd64.deb ...
Unpacking slirp4netns (1.0.1-2) ...
Setting up docker-buildx-plugin (0.14.0-1~ubuntu.22.04~jammy) ...
Setting up containerd.io (1.6.31-1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/containerd.service → /lib/systemd/system/containerd.service.
Setting up docker-compose-plugin (2.27.0-1~ubuntu.22.04~jammy) ...
Setting up docker-ce-cli (5:26.1.2-1~ubuntu.22.04~jammy) ...
Setting up libslirp0:amd64 (4.6.1-1build1) ...
Setting up pigz (2.6-1) ...
Setting up docker-ce-rootless-extras (5:26.1.2-1~ubuntu.22.04~jammy) ...
Setting up slirp4netns (1.0.1-2) ...
Setting up docker-ce (5:26.1.2-1~ubuntu.22.04~jammy) ...
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /lib/systemd/system/docker.service.
Created symlink /etc/systemd/system/sockets.target.wants/docker.socket → /lib/systemd/system/docker.socket.
Could not execute systemctl:  at /usr/bin/deb-systemd-invoke line 142.
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for libc-bin (2.35-0ubuntu3.7) ...
```

Aftet this the following packages are newly installed,
from Docker's official distribution:

```
# for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras; do \
   dpkg -l $pkg 2>/dev/null | grep ^ii\
done
ii  docker-ce      5:26.1.2-1~ubuntu.22.04~jammy amd64        Docker: the open-source application container engine
ii  docker-ce-cli  5:26.1.2-1~ubuntu.22.04~jammy amd64        Docker CLI: the open-source application container engine
ii  containerd.io  1.6.31-1     amd64        An open and reliable container runtime
ii  docker-buildx-plugin 0.14.0-1~ubuntu.22.04~jammy amd64        Docker Buildx cli plugin.
ii  docker-compose-plugin 2.27.0-1~ubuntu.22.04~jammy amd64        Docker Compose (V2) plugin for the Docker CLI.
ii  docker-ce-rootless-extras 5:26.1.2-1~ubuntu.22.04~jammy amd64        Rootless support for Docker.
```

**However**, *we're not done yet*; Docker Engine is
**not actually working** at this point.

Trying to verify that the Docker Engine installation is
successful by running the `hello-world` image, turns out
the Docker daemon is not yet running:

```
# docker run hello-world
docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?.
See 'docker run --help'.
```

Indeed the service is not running:

```
# systemctl status docker
× docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: failed (Result: exit-code) since Sun 2024-05-12 09:18:02 CEST; 5min ago
TriggeredBy: × docker.socket
       Docs: https://docs.docker.com
    Process: 1402955 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock (code=exited, status=1/FAILURE)
   Main PID: 1402955 (code=exited, status=1/FAILURE)
        CPU: 41ms

May 12 09:18:02 rapture systemd[1]: docker.service: Scheduled restart job, restart counter is at 3.
May 12 09:18:02 rapture systemd[1]: Stopped Docker Application Container Engine.
May 12 09:18:02 rapture systemd[1]: docker.service: Start request repeated too quickly.
May 12 09:18:02 rapture systemd[1]: docker.service: Failed with result 'exit-code'.
May 12 09:18:02 rapture systemd[1]: Failed to start Docker Application Container Engine.

# ls -l /run/containerd/containerd.sock /var/run/docker.sock
srw-rw---- 1 root root   0 May 12 09:17 /run/containerd/containerd.sock
srw-rw---- 1 root docker 0 May 11 22:29 /var/run/docker.sock
```

Both sockets exist, but apparently `dockerd` has a hard
time start the firs time and needs
[to be kicked up](https://github.com/docker/for-linux/issues/535):

```
# systemctl restart docker
# systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2024-05-12 09:33:02 CEST; 1s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1690331 (dockerd)
      Tasks: 22
     Memory: 30.6M
        CPU: 152ms
     CGroup: /system.slice/docker.service
             └─1690331 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

# docker version
Client: Docker Engine - Community
 Version:           26.1.2
 API version:       1.45
 Go version:        go1.21.10
 Git commit:        211e74b
 Built:             Wed May  8 13:59:59 2024
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          26.1.2
  API version:      1.45 (minimum version 1.24)
  Go version:       go1.21.10
  Git commit:       ef1912d
  Built:            Wed May  8 13:59:59 2024
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.31
  GitCommit:        e377cd56a71523140ca6ae87e30244719194a521
 runc:
  Version:          1.1.12
  GitCommit:        v1.1.12-0-g51d5e94
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

Now, finally, the `hello-world` example does work:

```
# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete 
Digest: sha256:a26bff933ddc26d5cdf7faa98b4ae1e3ec20c4985e6f87ac0973052224d24302
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

# docker images 
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
hello-world   latest    d2c94e258dcb   12 months ago   13.3kB
```

#### Configuration tweaks for containerd and Docker

With `containerd` installed and running, the next step is
to update its configuration to

*   Enable the use of [`systemd` cgroup driver](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd).
*   Enable CRI integration, which is disabled by default
    when installing `containerd` from Ubuntu packages,
    but is needed to use `containerd` with Kubernetes.
*   Move `/var/lib/containerd` to a bigger file system,
    this PC has most disk space mounted under `/home`.

```
# systemctl stop containerd
# cp -a /var/lib/containerd/ /home/lib/
# containerd config default \
 | sed 's:root = "/var/lib/:root = "/home/lib/:' \
 | sed 's/disabled_plugins.*/disabled_plugins = []/' \
 | sed 's/SystemdCgroup = false/SystemdCgroup = true/' \
 > /etc/containerd/config.toml
# mv /var/lib/containerd  /var/lib/DO_NOT_USE_containerd
# systemctl start containerd
```

**WARNING:** to create a **good**
`/etc/containerd/config.toml` one **must** do so by
modifying the **default** configuration. Using the
running configuration (with `containerd config dump`)
will produce an **invalid configuration** with an empty
`runtime_type` where that is not allowed. This leads to
[containerd issue #6964: default configuration isn't working
](https://github.com/containerd/containerd/issues/6964),
which makes `kubelet` failing to running any images,
and `kubeadm` fails to initialize the control pane.

**Note:** despite a misleading note in the 1.26
documentation, there is no need to manually configure the 
[cgroup driver for kubelet](https://v1-26.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/#configuring-the-kubelet-cgroup-driver).
because, already since v1.22,
**`kubeadm` defaults it to `systemd`**.

This would be a good time to do the same with 
`/var/lib/docker` and, when using BTRFS, add this
[Important Tweak for BTRFS]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html#important-tweak-for-btrfs).

Fist, create `/etc/docker/daemon.json` with this:

```json
{
  "data-root": "/home/lib/docker",
  "storage-driver": "overlay2"
}
```

Then, stop all `docker` and `containerd` services,
move `/var/lib/docker` and start services again:

```
# docker info | grep -E 'Storage Driver|Docker Root Dir'
 Storage Driver: overlay2
 Docker Root Dir: /var/lib/docker

# systemctl stop docker.service
# systemctl stop docker.socket
# systemctl stop containerd.service

# cp -a /var/lib/docker/ /home/lib/
# mv /var/lib/docker /var/lib/DO_NOT_USE_docker

# systemctl start docker.socket
# systemctl start docker.service
# systemctl start containerd.service

# docker info | grep -E 'Storage Driver|Docker Root Dir'
 Storage Driver: overlay2
 Docker Root Dir: /home/lib/docker
```

Possibly because Docker was already using `overlay2`,
possibly because the root partition is not using BTRFS,
Docker images were not lost as
[it happened in Lexicon]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html#ask-me-how-i-know).

**Warning:** at the end of the
[Container runtimes](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/)
documentation, it is suggested that the next step is to
install a [network plugin](https://v1-26.docs.kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model).
However, this should not be done before
**bootstrapping** the cluster (see next).

## Bootstrap the cluster with `kubeadm`

Having already
[installed kubeadm, kubelet and kubectl](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
the next step is to initialize the cluster:
[Creating acluster with kubeadm](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

### Initialize control-plane

Having reviewed the requirements and having installed all
the components already (see above), we proceed to
[Initializing your control-plane node](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node) with:

*  `--cri-socket=/run/containerd/containerd.sock` to make
   sure Kubernetes uses the `containerd` runtime.
*  `--pod-network-cidr=10.244.0.0/16` as
   [required by flannel](https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md),
   which is the [network plugin](#network-plugin) to be
   installed later.

Note that before initializing the control panel,
the `kubelet` service is not running:

```
# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Sun 2024-05-12 11:57:52 CEST; 4s ago
       Docs: https://kubernetes.io/docs/
    Process: 243927 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=1/FAILURE)
   Main PID: 243927 (code=exited, status=1/FAILURE)
        CPU: 43ms

# kubeadm init \
  --cri-socket=/run/containerd/containerd.sock \
  --pod-network-cidr=10.244.0.0/16

W0512 12:51:35.811501 1168278 initconfiguration.go:119] Usage of CRI endpoints without URL scheme is deprecated and can cause kubelet errors in the future. Automatically prepending scheme "unix" to the "criSocket" with value "/run/containerd/containerd.sock". Please update your configuration!
I0512 12:51:36.359676 1168278 version.go:256] remote version is much newer: v1.30.0; falling back to: stable-1.26
[init] Using Kubernetes version: v1.26.15
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local rapture] and IPs [10.96.0.1 10.0.0.2]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost rapture] and IPs [10.0.0.2 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost rapture] and IPs [10.0.0.2 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 5.501819 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node rapture as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node rapture as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: wjnord.g9lpm4heieilydn7
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.2:6443 --token wjnord.g9lpm4heieilydn7 \
        --discovery-token-ca-cert-hash sha256:bfe7582cfdf2d60c74c18da6aedfa7d5943b314cedd0ee25af996aecbd5a0c0f 
```

Now `kubelet` is running, if only reporting that
**Container runtime network not ready**:

```
# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Sun 2024-05-12 12:52:00 CEST; 3min 27s ago
       Docs: https://kubernetes.io/docs/
   Main PID: 1176576 (kubelet)
      Tasks: 23 (limit: 38142)
     Memory: 40.8M
        CPU: 3.311s
     CGroup: /system.slice/kubelet.service
             └─1176576 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:///run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.9

May 12 12:54:41 rapture kubelet[1176576]: E0512 12:54:41.015172 1176576 kubelet.go:2475] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
```

We can see all the containers already running:

```
# crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a 
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
d87c94df74a28       6c84132270a33       5 minutes ago       Running             kube-proxy                0                   51688b22b87e1       kube-proxy-tszr7
7513bd021911b       a0eed15eed449       5 minutes ago       Running             etcd                      0                   96b9a9ca6948e       etcd-rapture
641df12c76b68       5626d764f1326       5 minutes ago       Running             kube-scheduler            0                   6fd11b2b44804       kube-scheduler-rapture
ef7ad9f56205e       61669ae28d85c       5 minutes ago       Running             kube-apiserver            0                   e460715dc3a14       kube-apiserver-rapture
4244471dc3642       3b418eef6821c       5 minutes ago       Running             kube-controller-manager   0                   3f3e222d2483a       kube-controller-manager-rapture
```

Confirm the flag is sent in
`/var/lib/kubelet/kubeadm-flags.env`
to use the desired container runtime:

```
# grep container-runtime /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:/run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.9"
```

And check the cluster status. As `root` one can simply point
`KUBECONFIG` to the cluster's `admin.conf`:

```
# export KUBECONFIG=/etc/kubernetes/admin.conf
# kubectl cluster-info
Kubernetes control plane is running at https://10.0.0.6:6443
CoreDNS is running at https://10.0.0.6:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

To run `kubectl` as non-root, make a copy of that file under
your own `~/.kube` directory:

```
$ mkdir $HOME/.kube
$ sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ ls -l $HOME/.kube/config
-rw------- 1 coder coder 5659 Feb 19   2023 /home/coder/.kube/config
$ kubectl cluster-info
Kubernetes control plane is running at https://10.0.0.6:6443
CoreDNS is running at https://10.0.0.6:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

And *now* we can take care of the
*container runtime network*.

### Network plugin

[Installing a Pod network add-on](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)
is indeed the very next step after
*initializing the control-plane node*.

Once again, far from being clear and offering a
direction to follow, the documentation simply points to
[a non-exhaustive list of networking addons](https://v1-26.docs.kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
supported by Kubernetes, without much of a hint as to
which one/s may be a good choice for each scenario.

Having no desire to study multiple network plugins, lets
just install
[Flannel](https://github.com/flannel-io/flannel?tab=readme-ov-file#getting-started-on-kubernetes),
following the recommendation from 
[Computing for Geeks](https://computingforgeeks.com/install-kubernetes-cluster-ubuntu-jammy/) and
[LinuxConfig](https://linuxconfig.org/how-to-install-kubernetes-on-ubuntu-22-04-jammy-jellyfish-linux).

```
$ wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

$ kubectl apply -f kube-flannel.yml
namespace/kube-flannel created
serviceaccount/flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created

$ kubectl get all -n kube-flannel
NAME                        READY   STATUS    RESTARTS   AGE
pod/kube-flannel-ds-bbhn8   1/1     Running   0          14s

NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/kube-flannel-ds   1         1         1       1            1           <none>          14s
```

### Enable single-node cluster as Worker node

Before anything else is allowed to run in this cluster,
[Control plane node isolation](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation)
must be setup. By default, for security reasons,
the cluster will not schedule Pods to run in this node
because it is running the control plane. This is reflected
by the **Taints**:

```
$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
rapture   Ready    control-plane   38m   v1.26.15

$ kubectl describe node rapture
Name:               rapture
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=rapture
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"d6:46:a2:fe:00:f8"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.0.0.2
                    kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sun, 12 May 2024 12:51:57 +0200
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
...
```

Remove this taint to allow other pods to be scheduled:

```
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
node/rapture untainted

$ kubectl describe node rapture | grep -i taint
Taints:             <none>
```

#### Test Pod Scheduling

At this point the cluster is ready to run pods, here is
a simple test:

```
$ kubectl apply -f https://k8s.io/examples/pods/commands.yaml
pod/command-demo created

$ kubectl get all 
NAME               READY   STATUS      RESTARTS   AGE
pod/command-demo   0/1     Completed   0          10s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   64m

$ kubectl events pods
LAST SEEN   TYPE     REASON      OBJECT             MESSAGE
35m         Normal   NodeReady   Node/rapture       Node rapture status is now: NodeReady
20s         Normal   Scheduled   Pod/command-demo   Successfully assigned default/command-demo to rapture
19s         Normal   Pulling     Pod/command-demo   Pulling image "debian"
15s         Normal   Pulled      Pod/command-demo   Successfully pulled image "debian" in 4.165178746s (4.165185238s including waiting)
15s         Normal   Created     Pod/command-demo   Created container command-demo-container
15s         Normal   Started     Pod/command-demo   Started container command-demo-container
```

With the cluster now ready to run pods and services, we
can move on to installing more components that will be
used by the actual services:
[MetalLB Load Balancer](#metallb-load-balancer),
[Kubernets Dashboard](#kubernets-dashboard),
[Ingress Controller](#ingress-controller), with
[HTTPS with Let’s Encrypt](#https-with-lets-encrypt) and
a [LocalPath PV provisioner](#localpath-pv-provisioner)
for simple persistent storage in local file systems.

#### DO NOT `kubeadm join`

[Joining your nodes](https://v1-26.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)
is only to be done by **other** nodes, no the master.
The master node is *already joined* and `kubeadm`
[will not allow joining itself *again*](https://kodekloud.com/community/t/kubeadm-join-fails/337178/3).

Trying to do so, by running the `kubeadm join` command
provided by `kubeadm init` will fail:

```
# kubeadm join 10.0.0.2:6443 --token wjnord.g9lpm4heieilydn7 \
        --discovery-token-ca-cert-hash sha256:bfe7582cfdf2d60c74c18da6aedfa7d5943b314cedd0ee25af996aecbd5a0c0f

[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
        [ERROR Port-10250]: Port 10250 is in use
        [ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

In order to *resolve* those pre-flight checks one would
have to `kubeadm reset` which would only undo the work!

## MetalLB Load Balancer

A Load Balancer is going to be necessary for the
[Dashboard](#kubernets-dashboard)
and other services, to expose individual services via open
ports on the server (`NodePort`) or virtual IP addresses.
[Installation By Manifest](https://metallb.universe.tf/installation/#installation-by-manifest)
is as simple as applying the provided manifest:

```
$ wget https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

$ kubectl apply -f metallb-native.yaml
namespace/metallb-system created
customresourcedefinition.apiextensions.k8s.io/bfdprofiles.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bgpadvertisements.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bgppeers.metallb.io created
customresourcedefinition.apiextensions.k8s.io/communities.metallb.io created
customresourcedefinition.apiextensions.k8s.io/ipaddresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io created
customresourcedefinition.apiextensions.k8s.io/servicel2statuses.metallb.io created
serviceaccount/controller created
serviceaccount/speaker created
role.rbac.authorization.k8s.io/controller created
role.rbac.authorization.k8s.io/pod-lister created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/controller created
rolebinding.rbac.authorization.k8s.io/pod-lister created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
configmap/metallb-excludel2 created
secret/metallb-webhook-cert created
service/metallb-webhook-service created
deployment.apps/controller created
daemonset.apps/speaker created
validatingwebhookconfiguration.admissionregistration.k8s.io/metallb-webhook-configuration created
```

**Note:** YAML files for MetalLB will be stored for
future reference, under `1.26/metallb` in the
[GitHub repository](#github-repository).

MetalLB remains idle until configured, which is done by
deploying resources into its namespace (`metallb-system`).
In this PC, a small range of IP addresses is advertised via
[Layer 2 Configuration](https://metallb.universe.tf/configuration/#layer-2-configuration), which does not
not require the IPs to be bound to the network interfaces:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: rapture-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.230-192.168.1.240
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
```

Store this configuration in a separate, host-specific file (`ipaddress-pool-rapture.yaml`) and apply it:

```
$ kubectl apply -f ipaddress-pool-rapture.yaml
ipaddresspool.metallb.io/rapture-pool created
l2advertisement.metallb.io/l2-advert created

$ kubectl get l2advertisements.metallb.io -n metallb-system
NAME        IPADDRESSPOOLS   IPADDRESSPOOL SELECTORS   INTERFACES
l2-advert

$ kubectl get ipaddresspools.metallb.io -n metallb-system
NAME           AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
rapture-pool   true          false             ["192.168.1.230-192.168.1.240"]

$ kubectl describe ipaddresspools.metallb.io rapture-pool -n metallb-system
Name:         rapture-pool
Namespace:    metallb-system
Labels:       <none>
Annotations:  <none>
API Version:  metallb.io/v1beta1
Kind:         IPAddressPool
Metadata:
  Creation Timestamp:  2024-05-12T13:34:20Z
  Generation:          1
  Managed Fields:
    API Version:  metallb.io/v1beta1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:addresses:
        f:autoAssign:
        f:avoidBuggyIPs:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2024-05-12T13:34:20Z
  Resource Version:  13194
  UID:               979ae8fc-7cad-48df-9c2c-744c242a5075
Spec:
  Addresses:
    192.168.1.230-192.168.1.240
  Auto Assign:       true
  Avoid Buggy I Ps:  false
Events:              <none>
```

**Note:** The range 192.168.0.230-192.168.0.240 is
intended to be **outside** of the range leased by the
local DHCP server, which not only prevents conflicts but
also ensure that the router **will not** route external
requests to these.
This is the opposite intent from that of the 
[`IPAddressPool` setup in Lexicon]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html#metallb-load-balancer).

## Kubernets Dashboard

The first service to leverage all the above infrastructure,
including virtual IP addresses from the
[MetalLB Load Balancer](#metallb-load-balancer), is the
[Kubernetes Dashboard](https://v1-26.docs.kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/).

Once again, this service is deployed by manifest:

```
$ wget -O kubernetes-dashboard.yaml \
  https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

**Note:** YAML files for MetalLB will be stored for
future reference, under `1.26/dashboard` in the
[GitHub repository](#github-repository).

In this case to make the dashboard easily available in the
local network, we edit the deployment to add
`type: LoadBalancer` to the `kubernetes-dashboard`
**service** (line 40):

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

After applying this deployment, a virtual IP address will
be assign to it, where the dashboard can be accessed:

```
$ kubectl apply -f kubernetes-dashboard.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created

$ kubectl get svc -n kubernetes-dashboard
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP      10.107.131.104   <none>          8000/TCP        32s
kubernetes-dashboard        LoadBalancer   10.107.235.155   192.168.1.230   443:30480/TCP   32s
```

The dashboard is accessible at [https://192.168.1.230](https://192.168.1.230) but not exactly accessible without
a login token:

![Kubernetes Dashboard login page]({{ media }}/kubernetes-dashboard-login.png)

The [Authentication](kubernetes.io/docs/admin/authentication/)
link is broken; the documentation that seems most relevant to
authenticating Service Accounts to acess the dashboard is the
[Authenticating](https://v1-26.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/) page, which
does not quite explain how to create a Service Account,
grant it access to the dashboard *and then* obtain a token.
Neither does the documentation to
[Deploy **and Access** the Kubernetes Dashboard](https://v1-26.docs.kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/).

For this, I really had to resort to the
[Creating Kubernetes Admin User for Accessing Dashboard](https://computingforgeeks.com/create-admin-user-to-access-kubernetes-dashboard/)
article. To create a Service Account and grant it access to
everything (the `cluster-admin` roles), apply the following
manifest as `admin-sa-rbac.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8sadmin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  namespace: kube-system
  name: k8sadmin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: k8sadmin
    namespace: kube-system
```

```
$ kubectl apply -f admin-sa-rbac.yaml
serviceaccount/k8sadmin created
clusterrolebinding.rbac.authorization.k8s.io/k8sadmin created
```

Now one can create a token for the `k8sadmin` user and use it
to access the dashboard:

```
$ kubectl create token k8sadmin -n kube-system
eyJhbGciOiJSUzI1NiIsImtpZCI6IkktZ05rdlNzcVNzb1F0S3VYYmo3TXlpMjdTU2ktUUhpSzQwcUNLOTBRMXMifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzE1NTMwNTQyLCJpYXQiOjE3MTU1MjY5NDIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJrOHNhZG1pbiIsInVpZCI6IjJjMDliOTU3LTBhYzItNDQyNy05YjA2LWFiZDBhMmI5OTJhNyJ9fSwibmJmIjoxNzE1NTI2OTQyLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06azhzYWRtaW4ifQ.e6aAjUZvHvtZCN-QJAwFVt0I9pDklXC0La8RA-SjWTguEuAm0YbTgeOkRUKzb1q5_lkYIvpVyFN_E5z5IH1s6ygWdcWqS_RYtsmdQGr7iBTbpk-2cZb3bmt-mZkfDkYGgThNbzYoQ2gqXHSYnRtQKH2Spz6j_xt3G42IL1WGrfZIrdfBcLnkRBe1-l76qt1i0mq1wDwcSR2XAi3EpaOF-65f19__DYY4nt2JkhgxixpC-TZCLOXTfOAmbfbLoR07BmLZxw97Y85tvtYj1un7vQmgeDTBmWqtHIV4hYBlgH4ljUNIZg11GWECkOcd6UrHHtG8IG3REh87dAF2eD5Ysw
```

![Kubernetes Dashboard home page]({{ media }}/kubernetes-dashboard-home.png)

**Note:** at this point the SSL certificate is not yet valid,
this will be
[addressed later](#add-ingress-for-kubernetes-dashboard)
by accessing the dashboard via an
[Ingress Controller](#ingress-controller) with
[HTTPS](#https-with-lets-encrypt).

## Ingress Controller

An Nginx Ingress Controller will be used to redirect **HTTPS**
requests to different services depending on the `Host:` header,
while all those requests will be hitting the same IP address.

Download the deployment manifest following the
[Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/)
from
[kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)

```
$ wget -O nginx-ingress-controller.yaml \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

To serve HTTPS requests on a single IP address, the
`ingress-nginx-controller` **service** must be created with
`type: LoadBalancer`. As it happens, the `v1.10.1` manifest
already has this property set in line 366:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.10.1
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Local
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: LoadBalancer
```

After applying this deployment, another virtual IP address
is assigned to the `ingress-nginx-controller` service and
there is NGinx happily returning 404 Not found:

```
$ kubectl apply -f  nginx-ingress-controller.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created

$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.97.59.176   192.168.1.231   80:30706/TCP,443:30199/TCP   28s
ingress-nginx-controller-admission   ClusterIP      10.98.208.0    <none>          443/TCP                      28s

$ curl http://192.168.1.231/
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

## HTTPS with Let’s Encrypt

The
[Kubernetes setup for Let’s Encrypt in Lexicon]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html#kubernetes-setup)
was riddled with *first-time issues*, but I was able to
[Add Ingress for Kubernetes Dashboard]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html#add-ingress-for-kubernetes-dashboard)
and setup en entirely
[*automated* monthly renewal of certificates]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html#monthly-renewal-of-certificates-automated).
However, the entire *Let’s Encrypt* system to obtain and
renew certificates relies on external requests reaching the
NGinx controller, which is precisely some to avoid here.

If and when it becomes necessary, the setup can be replicated
even if that involves manually renewing certifidates each
month. While any other external requests would not be routed
to this PC, `/etc/hosts` can be used to map rapture.ssl.uu.am
to the NGinx LoadBalancer IP (192.168.1.231) so that
*external-looking* URLs can be used for this internal traffic.

In the meantime, `/etc/hosts` can be used to map specific
subdomains to the LoadBalancer IPs of individual services, e.g.
k8s.rapture.ssl.uu.am can be mapped to 192.168.1.230 so that
the dashboard is accessible at
[https://k8s.rapture.ssl.uu.am/](https://k8s.rapture.ssl.uu.am/).

## LocalPath PV provisioner

By default, a Kubernetes cluster is not set up to provide storage to pods. This was problematic for a while, especially
when trying to 
[deploy GitLab with Helm](https://docs.gitlab.com/charts/installation/deployment.html)
without first taking the time to
[Configure storage for the GitLab chart](https://docs.gitlab.com/charts/installation/storage.html).

Before finding out that the
[GitLab Helm chart](https://docs.gitlab.com/charts/) requires
more CPU to be allocated than available, I spent some time to
setup a
[LocalPath PV provisioner in Lexicon]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html#localpath-pv-provisioner)
but this was never really used. In the end, **all** services
deployed in Lexicon are using `storageClassName: manual` with
`hostPath` pointing to specific directories in local file
systems.

This may become necessary in the future, e.g. when installing
GitLab, at which point the setup can be replicated in this PC,
but that may be more appropriate to capture along with the work
to fullfil all other
[GitLab chart prerequisites](https://docs.gitlab.com/charts/installation/tools.html).