---
date: 2023-03-25
categories:
 - ubuntu
 - server
 - linux
 - kubernetes
 - docker
title: Single-node Kubernetes cluster on Ubuntu Server (lexicon)
---

After playing around with a few Docker containers and Docker
compose, I decided it was time to dive into Kubernetes.
But I only have one server:
[lexicon](2022-07-03-low-effort-homelab-server-with-ubuntu-server-on-intel-nuc.md).

For the most part I followed
[Computing for Geeks](https://computingforgeeks.com/)'
article
[Install Kubernetes Cluster on Ubuntu 22.04 using kubeadm](https://computingforgeeks.com/install-kubernetes-cluster-ubuntu-jammy/),
while taking some bits from
[How to install Kubernetes on Ubuntu 22.04 Jammy Jellyfish Linux](https://linuxconfig.org/how-to-install-kubernetes-on-ubuntu-22-04-jammy-jellyfish-linux)
(from [LinuxConfig](https://linuxconfig.org/))
and
[How to Install Kubernetes Cluster on Ubuntu 22.04](https://www.linuxtechi.com/install-kubernetes-on-ubuntu-22-04/)
(from [LinuxTechi](https://www.linuxtechi.com/)).

<!-- more --> 

## Docker

Since this was my first experience with containers, it all
started by
[installing docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
and
[playing around with it](https://docker-curriculum.com/).

Installing Docker is probably *not required* to the later
installation of Kubernetes, but it did influence my journey.

``` console
# apt update
# apt install -y ca-certificates curl gnupg lsb-release
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /usr/share/keyrings/docker.gpg
# arch=$(dpkg --print-architecture)
# signed_by=signed-by=/usr/share/keyrings/docker.gpg
# url=https://download.docker.com/linux/ubuntu
# echo "deb [arch=$arch $signed_by] $url $(lsb_release -cs) stable" \
  > /etc/apt/sources.list.d/docker.list > /dev/null
# apt update
# apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

To check the basic Docker components work:

``` console
# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete 
Digest: sha256:aa0cc8055b82dc2509bed2e19b275c8f463506616377219d9642221ab53cf9fe
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
```

### Storage Requirements

Docker images are store under `/var/lib/docker` by default,
this may not be a good location if the root partition is not
very big. In my case, most of the disk space is given to the
`/home` partition, so that's where I
[moved Docker images](https://github.com/IronicBadger/til/blob/master/docker/change-docker-root.md#option-3---createmodify-a-json-config-file-even-better-way).

First stop the Docker runtime service and move the files:

``` console
# systemctl stop docker.service
# systemctl stop docker.socket
# systemctl stop containerd.service
# mkdir /home/lib
# time cp -a /var/lib/docker/ /home/lib
# find /home/lib/docker \
  -name config.v2.json \
  -exec sed -i 's:/var/lib/:/home/lib/:g' {} \;
# containerd config default | sed 's:var/lib:home/lib:' \
> /etc/containerd/config.toml
```

Once files are moved, edit `/etc/docker/daemon.json` to point
the Docker runtime service to the new location:

``` json
{
  "data-root": "/home/lib/docker",
  ...
}
```

Finally, start the service again and check the change has
taken effect:

``` console
# systemctl start docker.socket
# systemctl start docker.service
# systemctl start containerd.service
# docker info | grep 'Docker Root Dir'
 Docker Root Dir: /home/lib/docker
```

#### Discard Unused Images

Over time, storage usage of `/home/lib/containerd`
will grow with stale images. These can be cleaned
up manually with `crictl rmi --prune` pointing it
to the correct endpoint:

``` console
# du -sh /home/lib/containerd/
109G    /home/lib/containerd/
# crictl \
  -r unix:///run/containerd/containerd.sock \
  rmi --prune
Deleted: registry.k8s.io/pause:3.9
# du -sh /home/lib/containerd/
8.1G    /home/lib/containerd/
```

The same can be achieved on an on-going basis by
adjusting the `ImageGCHighThresholdPercent` setting
to trigger Kubernetes' built-in garbage collection:

Setting this threshold involves
[updating the `KubeletConfiguration`](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure/#updating-the-kubeletconfiguration):

``` console
$ kubectl edit cm -n kube-system kubelet-config
```

Find the `imageMinimumGCAge` variable and add the
threshold for garbage collection under it; very
carefully using the exact same blank characters
for indentation (otherwise the YAML is invalid):

``` yaml
    imageMinimumGCAge: 0s
    imageGCLowThresholdPercent: 65
    imageGCHighThresholdPercent: 70
```

Run `kubeadm upgrade` as below to download the latest
`kubelet-config` ConfigMap contents into the local
file `/var/lib/kubelet/config.yaml` and restart the
`kubelet` service:

``` console
# kubeadm upgrade node phase kubelet-config
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.

# systemctl restart kubelet
```

!!! note

    This may not be effective when the images are not in the root partition.

[It also possible](https://github.com/containerd/containerd/discussions/6295#discussioncomment-1718081)
to set `discard_unpacked_layers: true` on the CRI
plugin configuration (`/etc/containerd/config.toml`)
so that it discards the compressed layer data after
unpacking images. This data is only really needed if
you are going to unpack the image again.

### Important Tweak for BTRFS

[Docker gradually exhausts disk space on BTRFS #27653](https://github.com/moby/moby/issues/27653).
This was a known issue since October **2016**,
and was closed in August 2023 with no plan to fix:

> There are no improvements planned in this area (BTRFS graph
> driver). containerd BTRFS is where future improvements
> might go. The overlay2 graph driver or overlayfs
> snapshotter is recommended for all users.

Docker still defaults to using its `btrfs` driver, so this
should be changed early on to avoid problems.

**Warning**: doing this later on will likely
result in losing all Docker images.
[Ask Me How I Know](#ask-me-how-i-know).

First stop the Docker runtime service and move the files:

``` console
# systemctl stop docker.service
# systemctl stop docker.socket
# systemctl stop containerd.service
```

Once files are moved, edit `/etc/docker/daemon.json` to
[override storage driver](https://github.com/moby/moby/issues/27653#issuecomment-1435749535)
as recommended:

``` json
{
  ...
  "storage-driver": "overlay2"
}
```

Finally, start the service again and check the change has
taken effect:

``` console
# systemctl start docker.socket
# systemctl start docker.service
# systemctl start containerd.service
# docker info | grep 'Storage Driver'
 Storage Driver: overlay2
```

#### Ask Me How I Know

I learned about the above issues the hard way, and it
resulted in losing all Docker images.

One day I found the root filesystem was nearly 100% full.

Pruning didn’t help much until I stopped all leftover clusters from previous exercises, and even then it only helped a bit:

``` console
# docker image prune -a -f
...
Total reclaimed space: 2.244GB
```

This reclaimed enough to bring
`/var/lib/docker/btrfs/subvolumes` down from 49G to 8.7G;
and the root filesystem went down to 78%.
At this point there were only 2 images running: code-server
and gitea. Stopped them, moved their volume’s `_data`
directories to another partition (under `/home/docker`),
updated their `docker-compose` files and it was all good.
Then stopped them again and removed all docker volumes,
except the only one that was in use.

With that, the root filesystem went down to 69%. Then used
this trick to remove all the subvolumes not in use; and got
the root filesystem down to 66%.
But all that was only a temporary relief; the partition
would quickly fill up again without overriding the storage
driver to use `overlay2`.

## Kubernetes

The core components of Kubernetes can be installed via APT:

``` console
# apt update
# apt full-upgrade
# curl -fsSLo \
  /etc/apt/keyrings/kubernetes-archive-keyring.gpg \
  https://packages.cloud.google.com/apt/doc/apt-key.gpg
# echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" \
  > /etc/apt/sources.list.d/kubernetes.list
# apt-get update
# apt-get install -y wget curl vim git kubelet kubeadm kubectl
# kubectl version --output=yaml
clientVersion:
  buildDate: "2023-03-15T13:40:17Z"
  compiler: gc
  gitCommit: 9e644106593f3f4aa98f8a84b23db5fa378900bd
  gitTreeState: clean
  gitVersion: v1.26.3
  goVersion: go1.19.7
  major: "1"
  minor: "26"
  platform: linux/amd64
kustomizeVersion: v4.5.7
```

The following command should be able to connect to the
Kubernetes `etcd` service now; you *might* need to run this
as root, or not:

``` console
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"26", GitVersion:"v1.26.1", GitCommit:"8f94681cd294aa8cfd3407b8191f6c70214973a4", GitTreeState:"clean", BuildDate:"2023-01-18T15:56:50Z", GoVersion:"go1.19.5", Compiler:"gc", Platform:"linux/amd64"}
```

### Networking Setup

The following steps were not necessary. Enabling IP forwarding and NAT is not necessary in Ubuntu 22.04 server, it is all already enabled by default apparently.

``` console
# modprobe overlay
# modprobe br_netfilter
# tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
# sysctl --system
```

The result, which can be checked upfront to confirm that the
above steps are not necessary, can be checked with:

``` console
# lsmod | egrep 'overlay|bridge'
bridge                307200  1 br_netfilter
stp                    16384  1 bridge
llc                    16384  2 bridge,stp
overlay               151552  0
# sysctl -a | egrep 'net.ipv4.ip_forward |net.bridge.bridge-nf-call-ip'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

### Migration to `containerd`

Normally, the next step would be to install a container
runtime, with the options being
*  [Docker](https://www.docker.com/), the *original* one.
*  [CRI-O](https://cri-o.io/), a *lightweight* option.
*  [containerd](https://containerd.io/), an industry-standard.

Docker and containerd were already installed as part of the
above first steps with Docker, Dockershim was deprecated in
Kubernetes 1.24 and the above step installed Kubernetes 1.26.

If we were already running Kubernetes (`kubelet`) with Docker
CE, at this point we’d have to point `kubelet` to containerd
by updating the flags in
`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` or 
`/var/lib/kubelet/kubeadm-flags.env` to

``` console
--container-runtime=remote 
--container-runtimeendpoint=unix:///run/containerd/containerd.sock"
```

But since we’re starting from scratch, we’ll do that from
`kubeadm` later:

``` console
# systemctl enable kubelet
# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Wed 2023-03-22 22:28:17 CET; 8s ago

# kubeadm config images pull \
  --cri-socket unix:/run/containerd/containerd.sock
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.26.3
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.26.3
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.26.3
[config/images] Pulled registry.k8s.io/kube-proxy:v1.26.3
[config/images] Pulled registry.k8s.io/pause:3.9
[config/images] Pulled registry.k8s.io/etcd:3.5.6-0
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.9.3
```

### Bootstrap

For a single-node cluster, bootstrap it in the simplest way:

``` console
# kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:/run/containerd/containerd.sock
[init] Using Kubernetes version: v1.26.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local lexicon] and IPs [10.96.0.1 10.0.0.6]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [lexicon localhost] and IPs [10.0.0.6 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [lexicon localhost] and IPs [10.0.0.6 127.0.0.1 ::1]
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
[apiclient] All control plane components are healthy after 6.505763 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node lexicon as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node lexicon as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: 8gotcn.nco65l9atfr0l77c
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

kubeadm join 10.0.0.6:6443 --token 8gotcn.nco65l9atfr0l77c \
        --discovery-token-ca-cert-hash \
  sha256:40002a46e8b15a41883d4d35fa68cef6ff2203fe79095da867b56a090abafa1a
```

Confirm the flag is sent in
`/var/lib/kubelet/kubeadm-flags.env`
to use the desired container runtime:

``` console
# grep container-runtime /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:/run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.9"
```

Check the customer status; as `root` one can simply point
`KUBECONFIG` to the cluster's `admin.conf`:

``` console
# export KUBECONFIG=/etc/kubernetes/admin.conf
# kubectl cluster-info
Kubernetes control plane is running at https://10.0.0.6:6443
CoreDNS is running at https://10.0.0.6:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

To run `kubectl` as non-root, make a copy of that file under
your own `~/.kube` directory:

``` console
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

#### Enable systemd cgroups

Containerd defauls to disabling systemd cgroups, which causes
the cluster to go into a crash-loop, making `kubectl` fail:

``` console
E0322 22:45:47.847433 1825724 memcache.go:265] couldn't get current server API group list:
Get "https://10.0.0.6:6443/api?timeout=32s": dial tcp 10.0.0.6:6443: connect: connection refused
```

In this scenario `etcd` will not be listening on port 6443 (check with `netstat -na`) and `journalctl` will show lots of
errors, starting with:

``` console
# journalctl -xe | egrep -i 'container|docker|kube|clust'
Mar 22 22:46:50 lexicon kubelet[1658461]: E0322 22:46:50.716943 1658461 pod_workers.go:965]
"Error syncing pod, skipping" err="failed to \"StartContainer\" for \"kube-apiserver\" with CrashLoopBackOff:
\"back-off 5m0s restarting failed container=kube-apiserver
pod=kube-apiserver-lexicon_kube-system(b003695d86b7fa8a74df14eff26934a5)\""
pod="kube-system/kube-apiserver-lexicon" podUID=b003695d86b7fa8a74df14eff26934a5
```

This `CrashLoopBackOff` error is a sign of 
[containerd issue #6009](https://github.com/containerd/containerd/issues/6009). To fix the problem,
[the workaround](https://github.com/containerd/containerd/issues/6009#issuecomment-1121672553)
is to set `SystemdCgroup = true` in
`/etc/containerd/config.toml` and restart the cluster with

``` console
# systemctl restart containerd
# systemctl restart kubelet
```

### Network Plugin

A Kubernetes cluster needs a simple layer 3 network for nodes
to communicate (even for a single-node cluster):

``` console
$ wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
$ kubectl apply -f kube-flannel.yml
namespace/kube-flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
$ kubectl get pods -n kube-flannel
NAME                    READY   STATUS    RESTARTS   AGE
kube-flannel-ds-nrrg6   1/1     Running   0          22s
```

### Add Worker Nodes

First, confirm the master node is ready:

``` console
$ kubectl get nodes -o wide
NAME      STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
lexicon   Ready    control-plane   9d    v1.26.3   10.0.0.6      <none>        Ubuntu 22.04.2 LTS   5.15.0-69-generic   containerd://1.6.19
```

At this point, for a
[single-node cluster](https://github.com/calebhailey/homelab/issues/3),
the command provided by `kubeadm init` is used to add the
same system as a worker:

``` console
# kubeadm join 10.0.0.6:6443 --token 8gotcn.nco65l9atfr0l77c \
        --discovery-token-ca-cert-hash \
  sha256:40002a46e8b15a41883d4d35fa68cef6ff2203fe79095da867b56a090abafa1a
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
        [ERROR Port-10250]: Port 10250 is in use
        [ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

**Note:** `kubeadm join` **must** be run as `root`.

At first the `kubeadm join` above fails because a Kubernetes
cluster *is not supposed* to have the same system work as
both master and worker.
This manifests as the node being *tainted*:

``` console
$ kubectl describe node lexicon
Name:               lexicon
Roles:              control-plane
...
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Unschedulable:      false
```

For a single-node cluster the only option appears to be
[Control plane node isolation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation)
as recommended in 
[github.com/calebhailey/homelab/issues/3](https://github.com/calebhailey/homelab/issues/3). After this, there is no taint:

``` console
$ kubectl taint nodes --all \
  node-role.kubernetes.io/control-plane-node/lexicon untainted

$ kubectl describe node lexicon
Name:               lexicon
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=lexicon
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"32:ae:4c:6b:5d:6f"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.0.0.6
                    kubeadm.alpha.kubernetes.io/cri-socket: unix:/run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 22 Mar 2023 22:37:43 +0100
Taints:             <none>
```

At this point the Kubernetes cluster is up and running and can
be tested deploying a test application:

``` console
$ kubectl apply -f https://k8s.io/examples/pods/commands.yaml

$ kubectl events pods
LAST SEEN   TYPE     REASON      OBJECT             MESSAGE
44m         Normal   NodeReady   Node/lexicon       Node lexicon status is now: NodeReady
27s         Normal   Scheduled   Pod/command-demo   Successfully assigned default/command-demo to lexicon
27s         Normal   Pulling     Pod/command-demo   Pulling image "debian"
21s         Normal   Pulled      Pod/command-demo   Successfully pulled image "debian" in 5.452640392s (5.452649743s including waiting)
21s         Normal   Created     Pod/command-demo   Created container command-demo-container
21s         Normal   Started     Pod/command-demo   Started container command-demo-container
```

### MetalLB Load Balancer

The [MetalLB Load Balancer](https://computingforgeeks.com/deploy-metallb-load-balancer-on-kubernetes/)
is going to be necessary for the [Dashboard](#dashboard)
and future applications, to expose individual services via
open ports on the server (`NodePort`) or virtual IP addresses.

``` console
$ wget \
https://raw.githubusercontent.com/metallb/metallb/v$MetalLB_RTAG/config/manifests/metallb-native.yaml
$ kubectl apply -f metallb-native.yaml
namespace/metallb-system created
customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bfdprofiles.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bgpadvertisements.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bgppeers.metallb.io created
customresourcedefinition.apiextensions.k8s.io/communities.metallb.io created
customresourcedefinition.apiextensions.k8s.io/ipaddresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io created
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
secret/webhook-server-cert created
service/webhook-service created
deployment.apps/controller created
daemonset.apps/speaker created
validatingwebhookconfiguration.admissionregistration.k8s.io/metallb-webhook-configuration created
```

After a few seconds the deployment should have the controller
and speaker running:

``` console
$ kubectl get all -n metallb-system 
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-68bf958bf9-kzcfg   1/1     Running   0          32s
pod/speaker-78bvh                 1/1     Running   0          32s

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/webhook-service   ClusterIP   10.96.89.141   <none>        443/TCP   32s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   1         1         1       1            1           kubernetes.io/os=linux   32s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           32s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-68bf958bf9   1         1         1       32s
$ kubectl get pods -n metallb-system 
NAME                          READY   STATUS    RESTARTS   AGE
controller-68bf958bf9-kzcfg   1/1     Running   0          92s
speaker-78bvh                 1/1     Running   0          92s
```

At this point the components are idle waiting for a working
[configuration](https://metallb.universe.tf/configuration/);
edit `ipaddress_pools.yaml` to set a range of IP addresses:

``` yaml linenums="1" title="ipaddress_pools.yaml"
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.122-192.168.0.140
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
```

The range **192.168.0.122-192.168.0.140** is based on the
local DHCP server being configured to lease 228 addresses
starting with 192.168.0.2. The current active leases are
reserved so they don’t change, and the range 122-140 are
just not leased so far. The reason to use IPs from the leased
range is that the router only allows adding port forwarding
rules for those. This range is intentionally on the same
network range and subnet as the DHCP server so that no
routing is required to reach MetalLB IP addresses.

**Note:** Layer 2 mode does not require the IPs to be bound
to the network interfaces of your worker nodes. It works by
responding to ARP requests on your local network directly,
to give the machine’s MAC address to clients.

``` console
$ kubectl apply -f ipaddress_pools.yaml 
ipaddresspool.metallb.io/production created
l2advertisement.metallb.io/l2-advert created
$ kubectl get ipaddresspool.metallb.io -n metallb-system
NAME         AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
production   true          false             ["192.168.0.122-192.168.0.140"]
$ kubectl get l2advertisement.metallb.io -n metallb-system
NAME        IPADDRESSPOOLS   IPADDRESSPOOL SELECTORS   INTERFACES
l2-advert                                              
$ kubectl describe ipaddresspool.metallb.io production -n metallb-system
Name:         production
Namespace:    metallb-system
API Version:  metallb.io/v1beta1
Kind:         IPAddressPool
Spec:
  Addresses:
    192.168.0.122-192.168.0.140
```

#### Test MetalLB

To test the load balancer with a demo web, create a 
deployment with `web-app-demo.yaml` as follows:

``` yaml linenums="1" title="web-app-demo.yaml"
apiVersion: v1
kind: Namespace
metadata:
  name: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
  namespace: web
spec:
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: httpd
        image: httpd:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-server-service
  namespace: web
spec:
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

``` console
$ kubectl apply -f web-app-demo.yaml
$ kubectl get all -n web
NAME                              READY   STATUS    RESTARTS   AGE
pod/web-server-5879949fb7-4c6r9   1/1     Running   0          26s

NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
service/web-server-service   LoadBalancer   10.111.32.131   192.168.0.122   80:32468/TCP   3s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web-server   1/1     1            1           26s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/web-server-5879949fb7   1         1         1       26s

$ curl http://192.168.0.122/
<html><body><h1>It works!</h1></body></html>
```

This only works from other hosts in the local network when
using an IP range in the same subnet, otherwise requests will
time out.

Also, this is not enough to make this IP’s port 80 reachable
from other networks. Adding a port forwarding rule in the
router to redirect port 12080 externally to port 80 on
192.168.0.122  works in that requests are forwarded to the
right port, but then requests are rejected.

### Ingress Controller

In addition to the load balancer, I also wanted to
[deploy Nginx Ingress Controller](https://computingforgeeks.com/deploy-nginx-ingress-controller-on-kubernetes-using-helm-chart/)
to redirect HTTPS requests to different services.

``` console
$ controller_tag=$(curl -s https://api.github.com/repos/kubernetes/ingress-nginx/releases/latest | grep tag_name | cut -d '"' -f 4)
$ wget -O nginx-ingress-controller-deploy.yaml \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/${controller_tag}/deploy/static/provider/baremetal/deploy.yaml
$ sed -i 's/type: NodePort/type: LoadBalancer/g' nginx-ingress-controller-deploy.yaml
```

**Note:** since we already have [MetalLB](#metallb-load-balancer)
configured with a range of IP addresses, change **line 365**
in `nginx-ingress-controller-deploy.yaml` to
`type: LoadBalancer` so that it gets an External IP.

``` console
$ kubectl apply -f nginx-ingress-controller-deploy.yaml
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
$ kubectl get all -n ingress-nginx
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-wjzv4        0/1     Completed   0          107s
pod/ingress-nginx-admission-patch-lcj2j         0/1     Completed   1          107s
pod/ingress-nginx-controller-6b58ffdc97-2k2hk   1/1     Running     0          107s

NAME                                         TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.100.229.186   192.168.0.122   80:31137/TCP,443:31838/TCP   6s
service/ingress-nginx-controller-admission   ClusterIP      10.99.206.192    <none>          443/TCP                      6s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           107s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-6b58ffdc97   1         1         1       107s

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           7s         107s
job.batch/ingress-nginx-admission-patch    1/1           8s         107s
```

### Dashboard

[Installing the Kubernetes Dashboard](https://computingforgeeks.com/how-to-install-kubernetes-dashboard-with-nodeport/)

Download the manifest and change spec.type to LoadBalancer so it gets an External IP from MetalLB:

``` console
$ VER=$(curl -s https://api.github.com/repos/kubernetes/dashboard/releases/latest|grep tag_name|cut -d '"' -f 4)
$ echo $VER
v2.7.0
$ wget -O kubernetes-dashboard.yaml \
  https://raw.githubusercontent.com/kubernetes/dashboard/$VER/aio/deploy/recommended.yaml 
```

To make the dashboard easily available in the local network,
edit `kubernetes-dashboard.yaml` (around line 40) to set the
service to `LoadBalancer`:
 
``` yaml linenums="36" title="kubernetes-dashboard.yaml"
  namespace: kubernetes-dashboard
spec:
  type: LoadBalancer
  ports:
    - port: 443
```

Then deploy the Dashboard:

``` console
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
$ kubectl get all -n kubernetes-dashboard
NAME                                            READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-7bc864c59-6k5tm   1/1     Running   0          33s
pod/kubernetes-dashboard-6c7ccbcf87-qd5cg       1/1     Running   0          33s

NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP      10.98.87.20     <none>          8000/TCP        33s
service/kubernetes-dashboard        LoadBalancer   10.99.222.155   192.168.0.123   443:31490/TCP   33s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dashboard-metrics-scraper   1/1     1            1           33s
deployment.apps/kubernetes-dashboard        1/1     1            1           33s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/dashboard-metrics-scraper-7bc864c59   1         1         1       33s
replicaset.apps/kubernetes-dashboard-6c7ccbcf87       1         1         1       33s
```

At this the dashboard is available at https://192.168.0.123/

### LocalPath PV provisioner

By default, a Kubernetes cluster is not set up to provide
storage to pods. The recommended way is to use
[dynamic volume provisioning](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#dynamic),
which is not currently enabled: `kube-apiserver` is running
with only `--enable-admission-plugins=NodeRestriction` which
means we need to enable the `DefaultStorageClass`
[admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)
on the API server, by updating this flag in line
20 of `/etc/kubernetes/manifests/kube-apiserver.yaml`

Next, we need to create
[persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
and **mark a storage class as default**.
Gitlab recommends setting `reclaimPolicy` to `Retain`.

> The name of a `StorageClass` object is significant, and is
> how users can request a particular class. Administrators
> set the name and other parameters of a class when first
> creating `StorageClass` objects, and the objects cannot be
> updated once they are created.

To use
[Local](https://kubernetes.io/docs/concepts/storage/storage-classes/#local)
volumes, will support `WaitForFirstConsumer` with
pre-created `PersistentVolume` binding, but not dynamic
provisioning.

For an easy way, we use the *Local Path Provisioner* from
[rancher.io](https://www.rancher.com/)
with `/home/k8s/local-path-storage`

To add a default storage class, I followed
[Computing for Geeks](https://computingforgeeks.com/)'
article
[Dynamic hostPath PV Creation in Kubernetes using Local Path Provisioner](https://computingforgeeks.com/dynamic-hostpath-pv-creation-in-kubernetes-using-local-path-provisioner/).

First, the cluster needs to be enabled for dynamic volume
provisioning, which means making sure `kube-apiserver` is
running with
`--enable-admission-plugins=DefaultStorageClass` to enable
the `DefaultStorageClass` admission controller, by updating
this flag in line 20 of 
`/etc/kubernetes/manifests/kube-apiserver.yaml`
and restarting the `kubelet` service:

``` yaml linenums="12" hl_lines="20" title="kube-apiserver.yaml"
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.0.0.6
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction,DefaultStorageClass
```

``` console
# systemctl restart kubelet.service
```

Create a directory in the a file system where there is plenty of space:

``` console
# mkdir /home/k8s/local-path-storage
# chmod 1777 /home/k8s/local-path-storage
```

Create a deployment with Rancher’s PV provisioner, with the annotation to make it the default storage class and create a
deployment, e.g. `local-path-storage-as-default-class.yaml`

??? k8s "Kubernetes deployment: `local-path-storage-as-default-class.yaml`"

    ``` yaml linenums="1" title="local-path-storage-as-default-class.yaml"
    apiVersion: v1
    kind: Namespace
    metadata:
      name: local-path-storage

    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: local-path-provisioner-service-account
      namespace: local-path-storage

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: local-path-provisioner-role
    rules:
      - apiGroups: [ "" ]
        resources: [ "nodes", "persistentvolumeclaims", "configmaps" ]
        verbs: [ "get", "list", "watch" ]
      - apiGroups: [ "" ]
        resources: [ "endpoints", "persistentvolumes", "pods" ]
        verbs: [ "*" ]
      - apiGroups: [ "" ]
        resources: [ "events" ]
        verbs: [ "create", "patch" ]
      - apiGroups: [ "storage.k8s.io" ]
        resources: [ "storageclasses" ]
        verbs: [ "get", "list", "watch" ]

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: local-path-provisioner-bind
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: local-path-provisioner-role
    subjects:
      - kind: ServiceAccount
        name: local-path-provisioner-service-account
        namespace: local-path-storage

    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: local-path-provisioner
      namespace: local-path-storage
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: local-path-provisioner
      template:
        metadata:
          labels:
            app: local-path-provisioner
        spec:
          serviceAccountName: local-path-provisioner-service-account
          containers:
            - name: local-path-provisioner
              image: rancher/local-path-provisioner:master-head
              imagePullPolicy: IfNotPresent
              command:
                - local-path-provisioner
                - --debug
                - start
                - --config
                - /etc/config/config.json
              volumeMounts:
                - name: config-volume
                  mountPath: /etc/config/
              env:
                - name: POD_NAMESPACE
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.namespace
          volumes:
            - name: config-volume
              configMap:
                name: local-path-config

    ---
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-path
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: rancher.io/local-path
    allowVolumeExpansion: true
    volumeBindingMode: WaitForFirstConsumer
    reclaimPolicy: Retain

    ---
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: local-path-config
      namespace: local-path-storage
    data:
      config.json: |-
        {
                "nodePathMap":[
                {
                        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                        "paths":["/home/k8s/local-path-storage"]
                }
                ]
        }
      setup: |-
        #!/bin/sh
        set -eu
        mkdir -m 0777 -p "$VOL_DIR"
      teardown: |-
        #!/bin/sh
        set -eu
        rm -rf "$VOL_DIR"
      helperPod.yaml: |-
        apiVersion: v1
        kind: Pod
        metadata:
          name: helper-pod
        spec:
          containers:
          - name: helper-pod
            image: busybox
            imagePullPolicy: IfNotPresent
    ```

``` console
$ kubectl apply -f \
  local-path-storage-as-default-class.yaml
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
configmap/local-path-config created

$ kubectl get storageclass 
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Retain          WaitForFirstConsumer   true                   11s
```

### HTTPS with Let's Encrypt

Enabling HTTPS and configuring SSL certificates with
[Let's Encrypt](https://letsencrypt.org/getting-started/)
requires running
[Certbot](https://certbot.eff.org/instructions?ws=other&os=ubuntufocal)
to confirm/certify control of the web host. Since nothing
is listening on port 80, this should be pretty easy.
The certificate should be for a specific subdomain
(e.g. `ssl.uu.am`) which will be mapped to the IP,
then other subdomains redirect to specific ports.

#### Host OS setup

[Install certbot](https://www.server-world.info/en/note?os=Ubuntu_22.04&p=ssl&f=2)
and request a certificate for `ssl.uu.am`

``` console
# apt install -y certbot
# certbot certonly --standalone -d ssl.uu.am
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): root@uu.am
...

# ls -l /etc/letsencrypt/live/ssl.uu.am/
total 20
lrwxrwxrwx 1 root root  38 Feb 13 22:00 cert.pem -> ../../archive/ssl.uu.am/cert1.pem
lrwxrwxrwx 1 root root  39 Feb 13 22:00 chain.pem -> ../../archive/ssl.uu.am/chain1.pem
lrwxrwxrwx 1 root root  43 Feb 13 22:00 fullchain.pem -> ../../archive/ssl.uu.am/fullchain1.pem
lrwxrwxrwx 1 root root  41 Feb 13 22:00 privkey.pem -> ../../archive/ssl.uu.am/privkey1.pem
-rw-r--r-- 1 root root 692 Feb 13 22:00 README

# cat /etc/letsencrypt/live/ssl.uu.am/README
This directory contains your keys and certificates.

`privkey.pem`  : the private key for your certificate.
`fullchain.pem`: the certificate file used in most server software.
`chain.pem`    : used for OCSP stapling in Nginx >=1.3.7.
`cert.pem`     : will break many server configurations, and should not be used
                 without reading further documentation (see link below).

WARNING: DO NOT MOVE OR RENAME THESE FILES!
         Certbot expects these files to remain in this location in order
         to function properly!

We recommend not moving these files. For more information, see the Certbot
User Guide at https://certbot.eff.org/docs/using.html#where-are-my-certificates.
```

Actual files are in `/etc/letsencrypt/archive/ssl.uu.am/`
but have numbered names (e.g. cert1.pem) while the links in
`/etc/letsencrypt/live/ssl.uu.am/` will have constant file
names (e.g. `cert.pem`).

##### Renewal (automated)

The `certbot` package installs a service and a timer to
renew certificates every 30 days, trying 2x daily:

``` console
# systemctl status certbot.timer 
● certbot.timer - Run certbot twice daily
     Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2023-04-16 13:42:11 CEST; 2s ago
    Trigger: n/a
   Triggers: ● certbot.service

Apr 16 13:42:11 lexicon systemd[1]: Stopped Run certbot twice daily.
Apr 16 13:42:11 lexicon systemd[1]: Stopping Run certbot twice daily...
Apr 16 13:42:11 lexicon systemd[1]: Started Run certbot twice daily.

# systemctl status certbot.service 
● certbot.service - Certbot
     Loaded: loaded (/lib/systemd/system/certbot.service; static)
     Active: activating (start) since Sun 2023-04-16 13:42:11 CEST; 10s ago
TriggeredBy: ● certbot.timer
       Docs: file:///usr/share/doc/python-certbot-doc/html/index.html
             https://certbot.eff.org/docs
   Main PID: 2015732 (certbot)
      Tasks: 1 (limit: 37940)
     Memory: 32.2M
        CPU: 256ms
     CGroup: /system.slice/certbot.service
             └─2015732 /usr/bin/python3 /usr/bin/certbot -q renew

Apr 16 13:42:11 lexicon systemd[1]: Starting Certbot...
```

For this to work long-term, the external port 80 must be
forwarded to the hosts’ IP.
If this forward is removed at some point (e.g. forwarded
to another IP), the renewal will fail:

``` console
# systemctl status certbot.service 
× certbot.service - Certbot
     Loaded: loaded (/lib/systemd/system/certbot.service; static)
     Active: failed (Result: exit-code) since Sun 2023-04-16 04:53:14 CEST; 8h ago
TriggeredBy: ● certbot.timer
       Docs: file:///usr/share/doc/python-certbot-doc/html/index.html
             https://certbot.eff.org/docs
    Process: 279698 ExecStart=/usr/bin/certbot -q renew (code=exited, status=1/FAILURE)
   Main PID: 279698 (code=exited, status=1/FAILURE)
        CPU: 632ms

Apr 16 04:50:22 lexicon systemd[1]: Starting Certbot...
Apr 16 04:53:14 lexicon certbot[279698]: Failed to renew certificate ssl.uu.am with error: Some challenges have>
Apr 16 04:53:14 lexicon certbot[279698]: All renewals failed. The following certificates could not be renewed:
Apr 16 04:53:14 lexicon certbot[279698]:   /etc/letsencrypt/live/ssl.uu.am/fullchain.pem (failure)
Apr 16 04:53:14 lexicon certbot[279698]: 1 renew failure(s), 0 parse failure(s)
Apr 16 04:53:14 lexicon systemd[1]: certbot.service: Main process exited, code=exited, status=1/FAILURE
Apr 16 04:53:14 lexicon systemd[1]: certbot.service: Failed with result 'exit-code'.
Apr 16 04:53:14 lexicon systemd[1]: Failed to start Certbot.
```

To fix this, restore the port forwarding and
**restart the timer** (not the service):

``` console
# systemctl restart certbot.timer 
# systemctl status certbot.timer 
● certbot.timer - Run certbot twice daily
     Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2023-04-16 13:42:11 CEST; 2s ago
    Trigger: n/a
   Triggers: ● certbot.service
```

After a while (random delay under 10 minutes) there should be a lot of activity logged in
`/var/log/letsencrypt/letsencrypt.log`
ending with

```
2023-04-16 13:49:44,607:DEBUG:certbot._internal.renewal:no renewal failures
```

#### Kubernetes setup

Having a successful setup to
[Enable HTTPS with Let's Encrypt](#https-with-lets-encrypt),
the question is how to set up Nginx reverse proxy.

We have an [Ingress Controller](#ingress-controller)
already configured with a MetalLB IP 192.168.0.122
listening on ports 80 and 443. We can forward external port
443 to this IP so we have the same page/s on (not yet
secure)
[https://192.168.0.121/](https://192.168.0.121/)
and
[https://ssl.uu.am/](https://ssl.uu.am/)
(just Nginx 404 now).

Need to make 2 big changes to the Nginx controller:

1. [Install Let’s Encrypt certificate as a secret](#install-lets-encrypt-certificate-as-a-secret)
1. [Add Ingress for the first pod](#add-ingress-for-the-first-pod)

##### Install Let’s Encrypt certificate as a secret

Need to add a secret in the `ingress-nginx` namespace with
the Let’s Encrypt certificate. Otherwise,
if nothing else is found, Nginx will use the default
"Kubernetes Ingress Controller Fake Certificate".

How to add the certificate as a secret?

[How to Install Kubernetes Cert-Manager and Configure Let’s Encrypt](https://www.howtogeek.com/devops/how-to-install-kubernetes-cert-manager-and-configure-lets-encrypt/)
is based on
[Securing NGINX-ingress](https://cert-manager.io/v1.3-docs/tutorials/acme/ingress/)
which we could follow starting from
[Step 5 - Deploy Cert Manager](https://cert-manager.io/v1.3-docs/tutorials/acme/ingress/#step-5---deploy-cert-manager)
by installing directly in
[Kubernetes](https://cert-manager.io/v1.3-docs/installation/kubernetes/)
(with or without Helm).
Install the latest version of cert-manager using Helm:

``` console
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
$ helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0 \
  --set installCRDs=true
NAME: cert-manager
LAST DEPLOYED: Sun Apr 16 15:25:25 2023
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.11.0 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/

$ kubectl get all -n cert-manager
NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-64f9f45d6f-qx4hs              1/1     Running   0          4m29s
pod/cert-manager-cainjector-56bbdd5c47-ltjgx   1/1     Running   0          4m29s
pod/cert-manager-webhook-d4f4545d7-tf4l6       1/1     Running   0          4m29s

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.110.150.206   <none>        9402/TCP   4m29s
service/cert-manager-webhook   ClusterIP   10.107.245.49    <none>        443/TCP    4m29s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           4m29s
deployment.apps/cert-manager-cainjector   1/1     1            1           4m29s
deployment.apps/cert-manager-webhook      1/1     1            1           4m29s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-64f9f45d6f              1         1         1       4m29s
replicaset.apps/cert-manager-cainjector-56bbdd5c47   1         1         1       4m29s
replicaset.apps/cert-manager-webhook-d4f4545d7       1         1         1       4m29s
```

Everything seems to be running fine, but just in case
[verify the installation](https://cert-manager.io/v1.3-docs/installation/kubernetes/#verifying-the-installation)
by creating a test cert.

Assuming that works,
[install the Kubectl plugin](https://cert-manager.io/v1.3-docs/usage/kubectl-plugin/);
get the latest release from
[github.com/cert-manager/cert-manager](https://github.com/cert-manager/cert-manager/releases/)
to match the Helm chart installed.

``` console
$ curl -L -o kubectl-cert-manager.tar.gz \
  https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/kubectl-cert_manager-linux-amd64.tar.gz
$ tar xfz kubectl-cert-manager.tar.gz 
$ sudo install -m 755 kubectl-cert_manager /usr/local/bin/
$ kubectl cert-manager check api
The cert-manager API is ready
```

Create a `ClusterIssuer` which applies across all Ingress
resources in the cluster, by deploying the following
`cert-manager-issuer.yml`

``` yaml linenums="1" title="cert-manager-issuer.yml"
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: root@uu.ma
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

``` console
$ kubectl create -f cert-manager-issuer.yml
clusterissuer.cert-manager.io/letsencrypt-prod created
$ kubectl describe clusterissuer.cert-manager.io/letsencrypt-prod
Name:         letsencrypt-prod
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         ClusterIssuer
Metadata:
  Creation Timestamp:  2023-04-16T14:11:19Z
  Generation:          1
  Resource Version:    3171554
  UID:                 2176e44f-421a-498c-935b-34e71a21cae1
Spec:
  Acme:
    Email:            root@uu.ma
    Preferred Chain:  
    Private Key Secret Ref:
      Name:  letsencrypt-prod
    Server:  https://acme-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Class:  nginx
Status:
  Acme:
    Last Registered Email:  root@uu.ma
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/1063900577
  Conditions:
    Last Transition Time:  2023-04-16T14:11:20Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
```

##### Add Ingress for the first pod

Now that we have `cert-manager` running in the cluster,
lets see how to update Ingress resources to request a
production certificate. Do this by changing the value
of the `cert-manager.io/cluster-issuer` annotation to
`letsencrypt-prod` (the `ClusterIssuer` above).

The example that follows shows the *rather bumpy*
journe of adding HTTPS to a pod running Gitea.

One way to do this would be using kubectl to apply the
change in Gitea's own Ingress (`vi gitea-ingress.yml`):

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
    - host: "git.ssl.uu.am"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gitea-http
                port:
                  number: 3000
  tls:
    - secretName: tls-secret
      hosts:
        - "git.ssl.uu.am"
```

``` console
$ kubectl -n gitea apply -f gitea-ingress.yml
```

Better yet, it should be possible to incorporate this
into values passed to the Gitea Helm chart, applying
the changes to `gitea-values.yaml` as follows:

``` yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/tls-acme: "true"
  hosts:
    - host: git.ssl.uu.am
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: tls-secret
      hosts:
        - "git.ssl.uu.am"
```

``` console
$ helm install gitea gitea-charts/gitea \
  --create-namespace \
  --namespace=gitea \
  --values=gitea-values.yaml \
  --dry-run --debug
# Source: gitea/templates/gitea/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea
  labels:
    helm.sh/chart: gitea-8.1.0
    app: gitea
    app.kubernetes.io/name: gitea
    app.kubernetes.io/instance: gitea
    app.kubernetes.io/version: "1.19.1"
    version: "1.19.1"
    app.kubernetes.io/managed-by: Helm
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
spec:
  rules:
    - host: "git.ssl.uu.am"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gitea-http
                port:
                  number: 3000

$ helm install gitea gitea-charts/gitea \
  --create-namespace \
  --namespace=gitea \
  --values=gitea-values.yaml 
coalesce.go:175: warning: skipped value for memcached.initContainers: Not a table.
NAME: gitea
LAST DEPLOYED: Sun Apr 16 16:32:07 2023
NAMESPACE: gitea
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  http://git.ssl.uu.am/

$ kubectl get ingress -A
NAMESPACE   NAME    CLASS    HOSTS                ADDRESS    PORTS   AGE
gitea       gitea   <none>   git.ssl.uu.am   10.0.0.6   80      20m

$ kubectl get all -n gitea
NAME                                   READY   STATUS    RESTARTS   AGE
pod/gitea-0                            1/1     Running   0          10m
pod/gitea-memcached-6559f55668-s8nl6   1/1     Running   0          10m
pod/gitea-postgresql-0                 1/1     Running   0          10m

NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/gitea-http            NodePort    10.100.158.203   <none>        3000:30080/TCP   10m
service/gitea-memcached       ClusterIP   10.111.252.194   <none>        11211/TCP        10m
service/gitea-postgresql      ClusterIP   10.96.77.223     <none>        5432/TCP         10m
service/gitea-postgresql-hl   ClusterIP   None             <none>        5432/TCP         10m
service/gitea-ssh             NodePort    10.105.190.218   <none>        22:30022/TCP     10m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gitea-memcached   1/1     1            1           10m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/gitea-memcached-6559f55668   1         1         1       10m

NAME                                READY   AGE
statefulset.apps/gitea              1/1     10m
statefulset.apps/gitea-postgresql   1/1     10m
```

This doesn’t work: Ingress gitea doesn’t have a class
and points to port 80 (wrong, should be 3000) Nginx on
(external and node’s) port 443 hasn’t changed
(404 Not Found, fake cert).

Maybe the way to use this is to have Nginx
(somewhere else) point to this service’s port 80?

While we have Gitea already running,
we can try using kubectl to apply this change to
`gitea-ingress.yml`:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
    - host: "git.ssl.uu.am"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gitea-http
                port:
                  number: 3000
  tls:
    - secretName: tls-secret
      hosts:
        - "git.ssl.uu.am"
```

But this still doesn't work:

``` console
$ kubectl -n gitea apply -f gitea-ingress.yml
Error from server (BadRequest): error when creating "gitea-ingress.yml": admission webhook "validate.nginx.ingress.kubernetes.io" denied the request: host "git.ssl.uu.am" and path "/" is already defined in ingress gitea/gitea
```

So we have to not try to add this directly to the Helm
chart, to avoid this conflict. Once that’s removed,
and Gitea reinstall (w/ Helm), we’re back at having an
Ingress entity jus like the one above:

``` console
$ kubectl get ingress -A
NAMESPACE   NAME            CLASS   HOSTS                ADDRESS    PORTS   AGE
gitea       gitea-ingress   nginx   git.ssl.uu.am   10.0.0.6   80      56m
```

**Note:** creating `gitea-ingress` in the
`ingress-nginx` namespace won’t work because then
Nginx can’t find the gitea-http service, resulting in
HTTP 503 error at
[https://git.ssl.uu.am](https://git.ssl.uu.am/)

```
W0416 15:55:11.088261       7 controller.go:1044] Error obtaining Endpoints for Service "ingress-nginx/gitea-http": no object matching key "ingress-nginx/gitea-http" in local store
```

At this point it should be possible to hit the
external IP on port 443, which is forwarded to the
MetalLB IP, and get the Gitea home page at
[https://git.ssl.uu.am](https://git.ssl.uu.am/)

That partially works: we get to the Gitea service,
but still use the fake cert.

``` console
$ kubectl -n ingress-nginx get pods | grep ingress-nginx-controller
ingress-nginx-controller-6b58ffdc97-rt5lm   1/1     Running     1 (4d12h ago)   14d

$ kubectl -n ingress-nginx logs ingress-nginx-controller-6b58ffdc97-rt5lm | tail -10
2023/04/16 16:08:53 [crit] 4655#4655: *6295724 SSL_do_handshake() failed (SSL: error:0A00006C:SSL routines::bad key share) while SSL handshaking, client: 10.244.0.1, server: 0.0.0.0:443
10.244.0.1 - - [16/Apr/2023:16:49:42 +0000] "GET / HTTP/2.0" 200 13744 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36" 464 0.004 [gitea-gitea-http-3000] [] 10.244.0.252:3000 13757 0.003 200 1df33130aa3f23144c62d7eb91650dcc
```

Not sure whether the SSL error there is caused by,
or causing, the use of the fake cert.

``` console
$ kubectl -n gitea describe ingress gitea-ingress
Name:             gitea-ingress
Labels:           <none>
Namespace:        gitea
Address:          10.0.0.6
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host                Path  Backends
  ----                ----  --------
  git.ssl.uu.am  
                      /   gitea-http:3000 (10.244.0.252:3000)
Annotations:          cert-manager.io/cluster-issuer: letsencrypt-prod
Events:               <none>
```

[docs.bitnami.com/kubernetes/.../secure-ingress-resources](https://docs.bitnami.com/kubernetes/infrastructure/cert-manager/configuration/secure-ingress-resources/)
looks like perhaps the cert-manager.io/cluster-issuer:
`letsencrypt-prod` annotation needs to go in the
ingress.class instead of each ingress resource

At this point, we the annotation to `nginx-ingress-controller-deploy.yaml` under line 601,
and check the diff and apply:

``` yaml
  kind: IngressClass
  metadata:
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
```

``` console
$ kubectl diff -f nginx-ingress-controller-deploy.yaml 
...
 kind: IngressClass
 metadata:
   annotations:
+    cert-manager.io/cluster-issuer: letsencrypt-prod
...

$ kubectl apply -f nginx-ingress-controller-deploy.yaml
$ kubectl -n ingress-nginx describe ingressclass.networking.k8s.io/nginx
Name:         nginx
Labels:       app.kubernetes.io/component=controller
              app.kubernetes.io/instance=ingress-nginx
              app.kubernetes.io/name=ingress-nginx
              app.kubernetes.io/part-of=ingress-nginx
              app.kubernetes.io/version=1.7.0
Annotations:  cert-manager.io/cluster-issuer: letsencrypt-prod
Controller:   k8s.io/ingress-nginx
Events:       <none>
```

That doesn’t seem to make any difference, so we keep looking and find out we’re missing the tls section in `gitea-ingress.yml` which turned to be important:

``` yaml
  tls:
    - secretName: tls-secret
      hosts:
        - "git.ssl.uu.am"
```

``` console
$ kubectl -n gitea apply -f gitea/gitea-ingress.yml 
ingress.networking.k8s.io/gitea-ingress configured

$ kubectl -n gitea describe ingress gitea-ingress
Name:             gitea-ingress
Labels:           <none>
Namespace:        gitea
Address:          10.0.0.6
Ingress Class:    nginx
Default backend:  <default>
TLS:
  tls-secret terminates git.ssl.uu.am
Rules:
  Host                Path  Backends
  ----                ----  --------
  git.ssl.uu.am
                      /   gitea-http:3000 (10.244.0.252:3000)
Annotations:          cert-manager.io/cluster-issuer: letsencrypt-prod
Events:
  Type    Reason             Age                 From                       Message
  ----    ------             ----                ----                       -------
  Normal  Sync               3s (x3 over 3h17m)  nginx-ingress-controller   Scheduled for sync
  Normal  CreateCertificate  3s                  cert-manager-ingress-shim  Successfully created Certificate "tls-secret"
```

We still get the fake cert on
[https://git.ssl.uu.am](https://git.ssl.uu.am/)
(and it’s not the browser caching it).

The ingress controller is complaining that it can’t
find those tls-secret

``` console
$ kubectl -n ingress-nginx logs ingress-nginx-controller-6b58ffdc97-rt5lm | egrep -i 'ssl.*cert|ssl.*hand' | grep -v gitlab
I0412 04:07:33.044575       7 main.go:104] "SSL fake certificate created" file="/etc/ingress-controller/ssl/default-fake-certificate.pem"
I0412 04:07:33.075730       7 ssl.go:533] "loading tls certificate" path="/usr/local/certificates/cert" key="/usr/local/certificates/key"
2023/04/16 12:41:20 [crit] 3974#3974: *6093931 SSL_do_handshake() failed (SSL: error:0A00006C:SSL routines::bad key share) while SSL handshaking, client: 10.244.0.1, server: 0.0.0.0:443
2023/04/16 15:12:03 [crit] 4112#4112: *6240379 SSL_do_handshake() failed (SSL: error:0A00006C:SSL routines::bad key share) while SSL handshaking, client: 10.244.0.1, server: 0.0.0.0:443
2023/04/16 16:08:53 [crit] 4655#4655: *6295724 SSL_do_handshake() failed (SSL: error:0A00006C:SSL routines::bad key share) while SSL handshaking, client: 10.244.0.1, server: 0.0.0.0:443
W0416 19:13:29.051149       7 controller.go:1372] Error getting SSL certificate "gitea/tls-secret": local SSL certificate gitea/tls-secret was not found. Using default certificate
W0416 19:13:29.088408       7 backend_ssl.go:47] Error obtaining X.509 certificate: no object matching key "gitea/tls-secret" in local store
...
W0416 19:20:05.541182       7 controller.go:1372] Error getting SSL certificate "kubernetes-dashboard/tls-secret": local SSL certificate kubernetes-dashboard/tls-secret was not found. Using default certificate
W0416 19:20:05.573367       7 backend_ssl.go:47] Error obtaining X.509 certificate: no object matching key "kubernetes-dashboard/tls-secret" in local store
```

The secrets do exist, do they not contain valid
certificates?

``` console
$ kubectl -n gitea describe cert
Name:         tls-secret
Namespace:    gitea
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2023-04-16T19:13:29Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  gitea-ingress
    UID:                   6e4dfaeb-23d8-4a24-906e-c457431f14e5
  Resource Version:        3200901
  UID:                     cb10b560-974f-4533-9eff-c5ff1a8d3933
Spec:
  Dns Names:
    git.ssl.uu.am
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  tls-secret
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:        2023-04-16T19:13:29Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      True
    Type:                        Issuing
    Last Transition Time:        2023-04-16T19:13:29Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      False
    Type:                        Ready
  Next Private Key Secret Name:  tls-secret-84npl
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    12m   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  12m   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "tls-secret-84npl"
  Normal  Requested  12m   cert-manager-certificates-request-manager  Created new CertificateRequest resource "tls-secret-hm94m"

$ kubectl -n kubernetes-dashboard describe cert
Name:         tls-secret
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2023-04-16T19:20:05Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  kubernetes-dashboard-ingress
    UID:                   dd2d357c-6558-41fd-bb0c-5b7eb4dc36d5
  Resource Version:        3201584
  UID:                     09f28e55-bbfd-4b0b-a171-80502fa8c88b
Spec:
  Dns Names:
    k8s.ssl.uu.am
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  tls-secret
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:        2023-04-16T19:20:05Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      True
    Type:                        Issuing
    Last Transition Time:        2023-04-16T19:20:05Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      False
    Type:                        Ready
  Next Private Key Secret Name:  tls-secret-dbggr
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    6m    cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  6m    cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "tls-secret-dbggr"
  Normal  Requested  6m    cert-manager-certificates-request-manager  Created new CertificateRequest resource "tls-secret-8w9rt"
```

Looking at
[Issuer not found #3066](https://github.com/cert-manager/cert-manager/discussions/3066),
it seems we have in-flight orders for certificates,
but are still waiting to receive them:

``` console
$ kubectl -n gitea describe CertificateRequest tls-secret-hm94m
Name:         tls-secret-hm94m
Namespace:    gitea
Labels:       <none>
Annotations:  cert-manager.io/certificate-name: tls-secret
              cert-manager.io/certificate-revision: 1
              cert-manager.io/private-key-secret-name: tls-secret-84npl
API Version:  cert-manager.io/v1
Kind:         CertificateRequest
Metadata:
  Creation Timestamp:  2023-04-16T19:13:29Z
  Generate Name:       tls-secret-
  Generation:          1
  Owner References:
    API Version:           cert-manager.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Certificate
    Name:                  tls-secret
    UID:                   cb10b560-974f-4533-9eff-c5ff1a8d3933
  Resource Version:        3200917
  UID:                     bb31c94f-fd83-4f62-9edc-5de1cc883b0a
Spec:
  Extra:
    authentication.kubernetes.io/pod-name:
      cert-manager-64f9f45d6f-qx4hs
    authentication.kubernetes.io/pod-uid:
      75a75d95-2318-4fc2-a3fa-0be649a24a31
  Groups:
    system:serviceaccounts
    system:serviceaccounts:cert-manager
    system:authenticated
  Issuer Ref:
    Group:  cert-manager.io
    Kind:   ClusterIssuer
    Name:   letsencrypt-prod
  Request:  …S0K
  UID:      088d0ec3-f284-4387-893c-e4a0eaa223e9
  Usages:
    digital signature
    key encipherment
  Username:  system:serviceaccount:cert-manager:cert-manager
Status:
  Conditions:
    Last Transition Time:  2023-04-16T19:13:29Z
    Message:               Certificate request has been approved by cert-manager.io
    Reason:                cert-manager.io
    Status:                True
    Type:                  Approved
    Last Transition Time:  2023-04-16T19:13:29Z
    Message:               Waiting on certificate issuance from order gitea/tls-secret-hm94m-3966586170: "pending"
    Reason:                Pending
    Status:                False
    Type:                  Ready
Events:
  Type    Reason              Age   From                                                Message
  ----    ------              ----  ----                                                -------
  Normal  WaitingForApproval  19m   cert-manager-certificaterequests-issuer-vault       Not signing CertificateRequest until it is Approved
  Normal  WaitingForApproval  19m   cert-manager-certificaterequests-issuer-selfsigned  Not signing CertificateRequest until it is Approved
  Normal  WaitingForApproval  19m   cert-manager-certificaterequests-issuer-venafi      Not signing CertificateRequest until it is Approved
  Normal  WaitingForApproval  19m   cert-manager-certificaterequests-issuer-acme        Not signing CertificateRequest until it is Approved
  Normal  WaitingForApproval  19m   cert-manager-certificaterequests-issuer-ca          Not signing CertificateRequest until it is Approved
  Normal  cert-manager.io     19m   cert-manager-certificaterequests-approver           Certificate request has been approved by cert-manager.io
  Normal  OrderCreated        19m   cert-manager-certificaterequests-issuer-acme        Created Order resource gitea/tls-secret-hm94m-3966586170

$ kubectl -n kubernetes-dashboard describe CertificateRequest tls-secret-8w9rt
Name:         tls-secret-8w9rt
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  cert-manager.io/certificate-name: tls-secret
              cert-manager.io/certificate-revision: 1
              cert-manager.io/private-key-secret-name: tls-secret-dbggr
API Version:  cert-manager.io/v1
Kind:         CertificateRequest
Metadata:
  Creation Timestamp:  2023-04-16T19:20:05Z
  Generate Name:       tls-secret-
  Generation:          1
  Owner References:
    API Version:           cert-manager.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Certificate
    Name:                  tls-secret
    UID:                   09f28e55-bbfd-4b0b-a171-80502fa8c88b
  Resource Version:        3201600
  UID:                     37fe511f-f278-4145-84f4-22d54a08f23b
Spec:
  Extra:
    authentication.kubernetes.io/pod-name:
      cert-manager-64f9f45d6f-qx4hs
    authentication.kubernetes.io/pod-uid:
      75a75d95-2318-4fc2-a3fa-0be649a24a31
  Groups:
    system:serviceaccounts
    system:serviceaccounts:cert-manager
    system:authenticated
  Issuer Ref:
    Group:  cert-manager.io
    Kind:   ClusterIssuer
    Name:   letsencrypt-prod
  Request:  …S0K
  UID:      088d0ec3-f284-4387-893c-e4a0eaa223e9
  Usages:
    digital signature
    key encipherment
  Username:  system:serviceaccount:cert-manager:cert-manager
Status:
  Conditions:
    Last Transition Time:  2023-04-16T19:20:05Z
    Message:               Certificate request has been approved by cert-manager.io
    Reason:                cert-manager.io
    Status:                True
    Type:                  Approved
    Last Transition Time:  2023-04-16T19:20:05Z
    Message:               Waiting on certificate issuance from order kubernetes-dashboard/tls-secret-8w9rt-548771682: "pending"
    Reason:                Pending
    Status:                False
    Type:                  Ready
Events:
  Type    Reason              Age   From                                                Message
  ----    ------              ----  ----                                                -------
  Normal  WaitingForApproval  13m   cert-manager-certificaterequests-issuer-ca          Not signing CertificateRequest until it is Approved
  Normal  WaitingForApproval  13m   cert-manager-certificaterequests-issuer-acme        Not signing CertificateRequest until it is Approved
  Normal  WaitingForApproval  13m   cert-manager-certificaterequests-issuer-selfsigned  Not signing CertificateRequest until it is Approved
  Normal  WaitingForApproval  13m   cert-manager-certificaterequests-issuer-vault       Not signing CertificateRequest until it is Approved
  Normal  WaitingForApproval  13m   cert-manager-certificaterequests-issuer-venafi      Not signing CertificateRequest until it is Approved
  Normal  cert-manager.io     13m   cert-manager-certificaterequests-approver           Certificate request has been approved by cert-manager.io
  Normal  OrderCreated        13m   cert-manager-certificaterequests-issuer-acme        Created Order resource kubernetes-dashboard/tls-secret-8w9rt-548771682
  Normal  OrderPending        13m   cert-manager-certificaterequests-issuer-acme        Waiting on certificate issuance from order kubernetes-dashboard/tls-secret-8w9rt-548771682: ""

$ kubectl describe clusterissuer letsencrypt-prod
Name:         letsencrypt-prod
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         ClusterIssuer
Metadata:
  Creation Timestamp:  2023-04-16T14:11:19Z
  Generation:          1
  Resource Version:    3171554
  UID:                 2176e44f-421a-498c-935b-34e71a21cae1
Spec:
  Acme:
    Email:            root@uu.am
    Preferred Chain:  
    Private Key Secret Ref:
      Name:  letsencrypt-prod
    Server:  https://acme-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Class:  nginx
Status:
  Acme:
    Last Registered Email:  root@uu.am
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/1063900577
  Conditions:
    Last Transition Time:  2023-04-16T14:11:20Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

The orders are pending, possibly because the self-check / challenge is failing due to incomplete network setup, i.e. not enough port forwardings:

``` console
$ kubectl -n gitea get order tls-secret-hm94m-3966586170
NAME                          STATE     AGE
tls-secret-hm94m-3966586170   pending   31m
$ kubectl -n kubernetes-dashboard get order tls-secret-8w9rt-548771682
NAME                         STATE     AGE
tls-secret-8w9rt-548771682   pending   24m
```

[Troubleshooting Orders](https://cert-manager.io/docs/troubleshooting/acme/#2-troubleshooting-orders)
shows how to find the orders, their challenges and
why they are failing:

``` console
$ kubectl -n gitea describe order tls-secret-hm94m-3966586170 | tail -4
Events:
  Type    Reason   Age   From                 Message
  ----    ------   ----  ----                 -------
  Normal  Created  34m   cert-manager-orders  Created Challenge resource "tls-secret-hm94m-3966586170-1941395475" for domain "git.ssl.uu.am"
$ kubectl -n gitea describe challenge tls-secret-hm94m-3966586170-1941395475 | grep Reason:
  Reason:      Waiting for HTTP-01 challenge propagation: failed to perform self check GET request 'http://git.ssl.uu.am/.well-known/acme-challenge/cw9i3YCPZQmQXrofkEt4inKYr5x3fEc9wJ6R2ydpeAg': Get "http://git.ssl.uu.am/.well-known/acme-challenge/cw9i3YCPZQmQXrofkEt4inKYr5x3fEc9wJ6R2ydpeAg": dial tcp 217.162.57.64:80: connect: connection refused

$ kubectl -n kubernetes-dashboard describe order tls-secret-8w9rt-548771682 | tail -4
Events:
  Type    Reason   Age   From                 Message
  ----    ------   ----  ----                 -------
  Normal  Created  28m   cert-manager-orders  Created Challenge resource "tls-secret-8w9rt-548771682-3606431526" for domain "k8s.ssl.uu.am"
$ kubectl -n kubernetes-dashboard describe challenge tls-secret-8w9rt-548771682-3606431526 |grep Reason:
  Reason:      Waiting for HTTP-01 challenge propagation: failed to perform self check GET request 'http://k8s.ssl.uu.am/.well-known/acme-challenge/Eu-aIJIbu4gRAqgRW2ZcPpGyVk9ZmIBFy23zDQNr14s': Get "http://k8s.ssl.uu.am/.well-known/acme-challenge/Eu-aIJIbu4gRAqgRW2ZcPpGyVk9ZmIBFy23zDQNr14s": dial tcp 217.162.57.64:80: connect: connection refused
```

The question is where to redirect that port 80 to;
currently it points to the node’s port 80.
Before trying to sort out that port 80 forwarding,
[HTTP01 troubleshooting](https://cert-manager.io/docs/troubleshooting/acme/#http01-troubleshooting)
looks like we want to add 2 more annotations to the
Ingress resources in `gitea-ingress.yml` and
`kubernetes-dashboard-ingress.yaml`:

``` yaml
cert-manager.io/issue-temporary-certificate: "true"
acme.cert-manager.io/http01-edit-in-place: "true"
```

Add these and re-apply:

``` console
$ kubectl -n kubernetes-dashboard apply -f  dashboard/kubernetes-dashboard-ingress.yaml
ingress.networking.k8s.io/kubernetes-dashboard-ingress configured

$ kubectl -n gitea apply -f gitea-ingress.yml 
ingress.networking.k8s.io/gitea-ingress configured
```

That doesn’t help, at least not immediately,
so **let’s sort out that port forwarding**: 
the trick is to find each acme resolver and forward
external port 80 to its `NodePort`:

``` console
$ kubectl -n gitea get svc | grep acme
cm-acme-http-solver-wmmvk   NodePort    10.103.37.86     <none>        8089:31544/TCP   51m
```

More generally, for other cert managers:

``` console
$ kubectl get svc -A | grep acme
code-server              cm-acme-http-solver-8n9ks                               NodePort       10.101.32.112    <none>          8089:32220/TCP                                                                                  3d19h
kubernetes-dashboard     cm-acme-http-solver-b8l2x                               NodePort       10.100.2.141     <none>          8089:32328/TCP
```

After that port was made accessible, the challenge disappears and the order is **completed successfully**:

``` console
$ kubectl -n gitea describe order tls-secret-hm94m-3966586170 | tail -4
  Type    Reason    Age   From                 Message
  ----    ------    ----  ----                 -------
  Normal  Created   54m   cert-manager-orders  Created Challenge resource "tls-secret-hm94m-3966586170-1941395475" for domain "git.ssl.uu.am"
  Normal  Complete  82s   cert-manager-orders  Order completed successfully

$ kubectl -n gitea describe CertificateRequest tls-secret-hm94m
Name:         tls-secret-hm94m

Status:
  Certificate:  LS0…==
  Conditions:
    Last Transition Time:  2023-04-16T19:13:29Z
    Message:               Certificate request has been approved by cert-manager.io
    Reason:                cert-manager.io
    Status:                True
    Type:                  Approved
    Last Transition Time:  2023-04-16T20:06:31Z
    Message:               Certificate fetched from issuer successfully
    Reason:                Issued
    Status:                True
    Type:                  Ready
Events:
...
  Normal  CertificateIssued   2m35s  cert-manager-certificaterequests-issuer-acme        Certificate fetched from issuer successfully

$ kubectl -n gitea describe cert
Name:         tls-secret
Namespace:    gitea
...
Status:
  Conditions:
    Last Transition Time:  2023-04-16T20:06:31Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
...
Events:
...
  Normal  Issuing    5m    cert-manager-certificates-issuing          The certificate has been successfully issued
```

Now we have certificates, and a new problem!

``` console
$ kubectl -n ingress-nginx logs ingress-nginx-controller-6b58ffdc97-rt5lm | grep -C1 error:0A00006C:SSL
10.244.0.1 - - [16/Apr/2023:20:08:27 +0000] "GET / HTTP/1.1" 200 13779 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.0.0 Safari/537.36" 742 0.003 [gitea-gitea-http-3000] [] 10.244.0.252:3000 13757 0.003 200 2ec12a59924906d8edd0900ff547afeb
2023/04/16 20:08:32 [crit] 5747#5747: *6530552 SSL_do_handshake() failed (SSL: error:0A00006C:SSL routines::bad key share) while SSL handshaking, client: 10.244.0.1, server: 0.0.0.0:443
10.244.0.1 - - [16/Apr/2023:20:10:55 +0000] "GET / HTTP/2.0" 200 36815 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36" 599 0.020 [gitea-gitea-http-3000] [] 10.244.0.252:3000 36828 0.020 200 566dde9f9d34e65ae66f7b1ff4efe1cd
```

Maybe the only problem was impatience; left alone for a while and it works perfectly now 🙂

##### Add Ingress for Kubernetes Dashboard

Once the above works, we can do the same for the
[Kubernetes Dashboard](#dashboard):

``` console
$ kubectl -n kubernetes-dashboard get svc
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.96.35.216    <none>        8000/TCP        13d
kubernetes-dashboard        NodePort    10.99.147.241   <none>        443:32000/TCP   13d
```

Update `kubernetes-dashboard-ingress.yaml` like this:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "false"
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.244.0.0/16"
spec:
  ingressClassName: nginx
  rules:
    - host: "k8s.ssl.uu.am"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubernetes-dashboard
                port:
                  number: 8443
```

``` console
$ kubectl -n kubernetes-dashboard \
  apply -f kubernetes-dashboard-ingress.yaml 
ingress.networking.k8s.io/kubernetes-dashboard-ingress created

$ kubectl -n kubernetes-dashboard get ingress
NAME                           CLASS   HOSTS                ADDRESS    PORTS   AGE
kubernetes-dashboard-ingress   nginx   k8s.ssl.uu.am   10.0.0.6   80      3m50s

$ kubectl -n kubernetes-dashboard describe ingress/kubernetes-dashboard-ingress
Name:             kubernetes-dashboard-ingress
Labels:           <none>
Namespace:        kubernetes-dashboard
Address:          10.0.0.6
Ingress Class:    nginx
Default backend:  <default>
TLS:
  tls-secret terminates k8s.ssl.uu.am
Rules:
  Host                Path  Backends
  ----                ----  --------
  k8s.ssl.uu.am  
                      /   kubernetes-dashboard:8443 (10.244.0.82:8443)
Annotations:          cert-manager.io/cluster-issuer: letsencrypt-prod
                      nginx.ingress.kubernetes.io/auth-tls-verify-client: false
                      nginx.ingress.kubernetes.io/backend-protocol: HTTPS
                      nginx.ingress.kubernetes.io/whitelist-source-range: 10.244.0.0/16
Events:
  Type    Reason             Age                 From                       Message
  ----    ------             ----                ----                       -------
  Normal  Sync               16s (x6 over 123m)  nginx-ingress-controller   Scheduled for sync
  Normal  CreateCertificate  16s                 cert-manager-ingress-shim  Successfully created Certificate "tls-secret"

$ kubectl get all -A -o wide | egrep 10.244.0.82
kubernetes-dashboard   pod/kubernetes-dashboard-6c7ccbcf87-vwxnt       1/1     Running     1 (4d13h ago)    13d     10.244.0.82    lexicon   <none>           <none>
```

This does not yet work 100%:
- [ssl.uu.am:32000](https://ssl.uu.am:32000/#/workloads?namespace=_all) works,
  port 32000 is forwarded to the correct `NodePort`
- [k8s.uu.am](https://k8s.uu.am/) redirects there,
  with an HTTP redirect rule
- [k8s.ssl.uu.am/](https://k8s.ssl.uu.am/#/workloads?namespace=_all)
  doesn’t work (HTTP ERROR 400)
- `k8s.ssl.uu.am` resolves to external IP, where port
   443 is forwarded to the
   [MetalLB](#metallb-load-balancer) IP 192.168.0.122
   for `ingress-nginx-controller`.

Looking at the logs of the Nginx controller, it looks
like a problem with/in the Cluster internal network:

``` console
$ kubectl -n ingress-nginx logs ingress-nginx-controller-6b58ffdc97-rt5lm | tail -2
2023/04/16 17:26:40 [error] 4790#4790: *6371572 recv() failed (104: Connection reset by peer) while reading upstream, client: 10.244.0.1, server: k8s.ssl.uu.am, request: "GET / HTTP/2.0", upstream: "http://10.244.0.82:8443/", host: "k8s.ssl.uu.am", referrer: "https://www.google.com/"
10.244.0.1 - - [16/Apr/2023:17:26:40 +0000] "GET / HTTP/2.0" 400 48 "https://www.google.com/" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36" 465 0.002 [kubernetes-dashboard-kubernetes-dashboard-443] [] 10.244.0.82:8443 48 0.002 400 c2e45b0dee94255bb284054654b987a4
```

Tried mapping to port 8443 but still got the same error. The problem was that Nginx was trying to send HTTP requests to an HTTPS endpoint and the solution was to add more annotations in
`kubernetes-dashboard-ingress.yaml` and re-apply:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "false"
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.244.0.0/16"
```

[These additional annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#backend-certificate-authentication)
might be necessary later,
but it’s not clear when or why:

``` yaml
    nginx.ingress.kubernetes.io/configuration-snippet: |-
      proxy_ssl_server_name on;
      proxy_ssl_name $host;
```

In the end, what was missing was the `tls` section and
after troubleshooting challenges, we got it to work.

Finally, we can have [http://k8s.uu.am](http://k8s.uu.am)
redirect to 
[k8s.ssl.uu.am](https://k8s.ssl.uu.am/#/workloads?namespace=_all)
which not only works but is also entirely safe! 🙂

##### Monthly renewal of certificates (manual)

A [montly renewal of certificates](#renewal-automated)
is scheduled for Kubernetes deployments, but then again
it will create challenges that can't be fulfilled until
port 80 on the public IP is forwarded to the right
node port, *one at a time*.

This can be automated by having port 80 on the public
IP permanently forwarded to the host port 80, which is
never listening, and then redirecting that port to the
node port for each `acme` resolver, one at a time.

To find **all** `acme` resolvers:

``` console
$ kubectl get svc -A | grep acme
code-server              cm-acme-http-solver-8n9ks                               NodePort       10.101.32.112    <none>          8089:32220/TCP                                                                                  3d19h
kubernetes-dashboard     cm-acme-http-solver-b8l2x                               NodePort       10.100.2.141     <none>          8089:32328/TCP
```

In this situation, we'd want to forward port 32220 to
port 80 until the `acme` resolver listening on that
port is no longer running. Then forward port 32328 to
port 80 until the `acme` resolver listening on that
port is no longer running. And so on.

##### Monthly renewal of certificates (automated)

Changing port forwarding on the router is very slow,
so instead we want to
[change service ports using patch](https://stackoverflow.com/questions/65789509/kubernetes-how-to-change-service-ports-using-patch)
on those `cm-acme-http-solver` *services* to change
their `nodePort` to one the router is *always*
redirecting port 80 to.

Once the service is patched, the external connection
reaches the solver pod and the renewal is completed
very shortly, which then deletes the pod and service.

Running this in a crontab daily should do the job:

``` bash
#!/bin/bash
#
# Patch the nodePort of running cert-manager renewal challenge, to listen
# on port 32080 which is the one the router is forwarding port 80 to.

# Check if there is a LetsEncrypt challenge resolver (acme) running.
export KUBECONFIG=/etc/kubernetes/admin.conf
nodeport=$(kubectl get svc -A | grep acme | awk '{print $6}' | cut -f2 -d: | cut -f1 -d/ | head -1)
namespace=$(kubectl get svc -A | grep acme | awk '{print $1}' | head -1)
service=$(kubectl get svc -A | grep acme | awk '{print $2}' | head -1)

# Patch the service to listen on port 32080 (set up in router).
if [ -n "${namespace}" ] && [ -n "${service}" ]; then
    kubectl -n "${namespace}" patch service "${service}" -p '{"spec":{"ports": [{"port": 8089, "nodePort":32080}]}}'
fi
```

``` console
# crontab -e
# Hourly patch montly cert renewal solvers.
30 * * * * /root/bin/cert-renewal-port-fwd.sh
```
