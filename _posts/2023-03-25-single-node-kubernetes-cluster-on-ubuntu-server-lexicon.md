---
title:  "Single-node Kubernetes cluster on Ubuntu Server (lexicon)"
date:   2023-03-25 23:03:25 +0200
categories: ubuntu server linux kubernetes docker
---

After playing around with a few Docker containers and Docker
compose, I decided it was time to dive into Kubernetes.
But I only have one server:
[lexicon]({{ site.baseurl }}/2022/07/03/low-effort-homelab-server-with-ubuntu-server-on-intel-nuc.html).

{% assign media = site.baseurl | append: "/assets/media/" | append:  page.path | replace: ".md","" | replace: "_posts/",""  %}

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

## Docker

Since this was my first experience with containers, it all
started by
[installing docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
and
[playing around with it](https://docker-curriculum.com/).

Installing Docker is probably *not required* to the later
installation of Kubernetes, but it did influence my journey.

```
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

```
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

```
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

```js
{
  "data-root": "/home/lib/docker",
  ...
}
```

Finally, start the service again and check the change has
taken effect:

```
# systemctl start docker.socket
# systemctl start docker.service
# systemctl start containerd.service
# docker info | grep 'Docker Root Dir'
 Docker Root Dir: /home/lib/docker
```

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

```
# systemctl stop docker.service
# systemctl stop docker.socket
# systemctl stop containerd.service
```

Once files are moved, edit `/etc/docker/daemon.json` to
[override storage driver](https://github.com/moby/moby/issues/27653#issuecomment-1435749535)
as recommended:

```js
{
  ...
  "storage-driver": "overlay2"
}
```

Finally, start the service again and check the change has
taken effect:

```
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

```
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

```
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

```
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"26", GitVersion:"v1.26.1", GitCommit:"8f94681cd294aa8cfd3407b8191f6c70214973a4", GitTreeState:"clean", BuildDate:"2023-01-18T15:56:50Z", GoVersion:"go1.19.5", Compiler:"gc", Platform:"linux/amd64"}
```

### Networking Setup

The following steps were not necessary. Enabling IP forwarding and NAT is not necessary in Ubuntu 22.04 server, it is all already enabled by default apparently.

```
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

```
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

```
--container-runtime=remote 
--container-runtimeendpoint=unix:///run/containerd/containerd.sock"
```

But since we’re starting from scratch, we’ll do that from
`kubeadm` later:

```
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

```
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

```
# grep container-runtime /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:/run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.9"
```

Check the customer status; as `root` one can simply point
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

#### Enable systemd cgroups

Containerd defauls to disabling systemd cgroups, which causes
the cluster to go into a crash-loop, making `kubectl` fail:

```
E0322 22:45:47.847433 1825724 memcache.go:265] couldn't get current server API group list:
Get "https://10.0.0.6:6443/api?timeout=32s": dial tcp 10.0.0.6:6443: connect: connection refused
```

In this scenario `etcd` will not be listening on port 6443 (check with `netstat -na`) and `journalctl` will show lots of
errors, starting with:

```
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

```
# systemctl restart containerd
# systemctl restart kubelet
```

### Network Plugin

A Kubernetes cluster needs a simple layer 3 network for nodes
to communicate (even for a single-node cluster):

```
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

```
$ kubectl get nodes -o wide
NAME      STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
lexicon   Ready    control-plane   9d    v1.26.3   10.0.0.6      <none>        Ubuntu 22.04.2 LTS   5.15.0-69-generic   containerd://1.6.19
```

At this point, for a
[single-node cluster](https://github.com/calebhailey/homelab/issues/3),
the command provided by `kubeadm init` is used to add the
same system as a worker:

```
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

```
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

```
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

```
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

```
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

```
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

```yaml
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

```
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

```yaml
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

```
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

```
$ controller_tag=$(curl -s https://api.github.com/repos/kubernetes/ingress-nginx/releases/latest | grep tag_name | cut -d '"' -f 4)
$ wget -O nginx-ingress-controller-deploy.yaml \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/${controller_tag}/deploy/static/provider/baremetal/deploy.yaml
$ sed -i 's/type: NodePort/type: LoadBalancer/g' nginx-ingress-controller-deploy.yaml
```

**Note:** since we already have [MetalLB](#metallb-load-balancer)
configured with a range of IP addresses, change **line 365**
in `nginx-ingress-controller-deploy.yaml` to
`type: LoadBalancer` so that it gets an External IP.

```
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

```
$ VER=$(curl -s https://api.github.com/repos/kubernetes/dashboard/releases/latest|grep tag_name|cut -d '"' -f 4)
$ echo $VER
v2.7.0
$ wget -O kubernetes-dashboard.yaml \
  https://raw.githubusercontent.com/kubernetes/dashboard/$VER/aio/deploy/recommended.yaml 
```

To make the dashboard easily available in the local network,
edit `kubernetes-dashboard.yaml` (around line 40) to set the
service to `LoadBalancer`:
 
```yaml
  namespace: kubernetes-dashboard
spec:
  type: LoadBalancer
  ports:
    - port: 443
```

Then deploy the Dashboard:

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

### HTTPS with Let's Encrypt