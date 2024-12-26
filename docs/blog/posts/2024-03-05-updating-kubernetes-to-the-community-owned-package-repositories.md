---
date: 2024-03-05
categories:
 - linux
 - kubernetes
 - docker
 - server
 - ubuntu
title: Updating Kubernetes to the Community-Owned Package Repositories
---

## An Unexpected Update

Most days I update my little server when I log into my PC,
and today it gave quite an unexpected surprise:

``` console
# apt update && apt full-upgrade -y
...
E: The repository 'https://apt.kubernetes.io kubernetes-xenial Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

This was promptly reported already yesterday as
[Ubuntu kubernetes-xenial package repository issue #123673](https://github.com/kubernetes/kubernetes/issues/123673) and quick triaged
pointing to the announcement from August 2023:
[pkgs.k8s.io: Introducing Kubernetes Community-Owned Package Repositories](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/).

<!-- more --> 

Note that there is now a separate repository for each minor version,
so we need to first check which version is currently running:

``` console
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.2", GitCommit:"89a4ea3e1e4ddd7f7572286090359983e0387b2f", GitTreeState:"clean", BuildDate:"2023-09-13T09:34:32Z", GoVersion:"go1.20.8", Compiler:"gc", Platform:"linux/amd64"}
```

That we migrate to the **1.28** repository:

``` console
# echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" \
  | tee /etc/apt/sources.list.d/kubernetes.list
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# apt update && apt full-upgrade -y
```

This **replaced** the old repository
in `/etc/apt/sources.list.d/kubernetes.list`
that was installed in March,
when creating a [Single-node Kubernetes cluster on Ubuntu Server (lexicon)](https://stibbons1990.github.io/hex/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html):

```
deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main
```

## An Unexpected Breakage

Turns out updating to 1.28 *broke everything*: all services are unreachable. Can't read logs of pods either.

``` console
$ kubectl -n plexserver logs -f plexserver-85f7bf866-7c8gv
Error from server: Get "https://10.0.0.6:10250/containerLogs/plexserver/plexserver-85f7bf866-7c8gv/plexserver?follow=true": dial tcp 10.0.0.6:10250: connect: connection refused
$ netstat -na | grep 10250
(nothing)
```

Looking back at the process followed when creating the [Single-node Kubernetes cluster on Ubuntu Server (lexicon)](https://stibbons1990.github.io/hex/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html)
it was actually **1.26** that was installed, and it seems the update
above put the `clientVersion` up to 1.28 but not the `serverVersion`:

``` console
$ kubectl version --output=yaml
clientVersion:
  buildDate: "2024-02-14T10:40:48Z"
  compiler: gc
  gitCommit: c8dcb00be9961ec36d141d2e4103f85f92bcf291
  gitTreeState: clean
  gitVersion: v1.28.7
  goVersion: go1.21.7
  major: "1"
  minor: "28"
  platform: linux/amd64
kustomizeVersion: v5.0.4-0.20230601165947-6ce0bf390ce3
serverVersion:
  buildDate: "2023-03-15T13:33:12Z"
  compiler: gc
  gitCommit: 9e644106593f3f4aa98f8a84b23db5fa378900bd
  gitTreeState: clean
  gitVersion: v1.26.3
  goVersion: go1.19.7
  major: "1"
  minor: "26"
  platform: linux/amd64

WARNING: version difference between client (1.28) and server (1.26) exceeds the supported minor version skew of +/-1

$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.7", GitCommit:"c8dcb00be9961ec36d141d2e4103f85f92bcf291", GitTreeState:"clean", BuildDate:"2024-02-14T10:39:01Z", GoVersion:"go1.21.7", Compiler:"gc", Platform:"linux/amd64"}
```

Downgraded to 1.26 but that was not enough.

``` console
# echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.26/deb/ /" \
  | tee /etc/apt/sources.list.d/kubernetes.list
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.26/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# apt update
# apt-get install --reinstall \
  --allow-downgrades \
  cri-tools=1.26.0-1.1 \
  kubelet=1.26.3-1.1 \
  kubectl=1.26.3-1.1 \
  kubeadm=1.26.3-1.1
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages will be DOWNGRADED:
  cri-tools kubeadm kubectl kubelet
0 upgraded, 0 newly installed, 4 downgraded, 0 to remove and 0 not upgraded.
Need to get 59.3 MB of archives.
After this operation, 6,612 kB of additional disk space will be used.
Do you want to continue? [Y/n]
Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  kubelet 1.26.3-1.1 [20.5 MB]
Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  kubectl 1.26.3-1.1 [10.1 MB]
Get:3 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  kubeadm 1.26.3-1.1 [9,750 kB]
Get:4 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.26/deb  cri-tools 1.26.0-1.1 [19.0 MB]
Fetched 59.3 MB in 2s (34.9 MB/s)   
```

After this both versions are 1.26:

``` console
$ kubectl version --output=yaml
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
serverVersion:
  buildDate: "2023-03-15T13:33:12Z"
  compiler: gc
  gitCommit: 9e644106593f3f4aa98f8a84b23db5fa378900bd
  gitTreeState: clean
  gitVersion: v1.26.3
  goVersion: go1.19.7
  major: "1"
  minor: "26"
  platform: linux/amd64

$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"26", GitVersion:"v1.26.3", GitCommit:"9e644106593f3f4aa98f8a84b23db5fa378900bd", GitTreeState:"clean", BuildDate:"2023-03-15T13:38:47Z", GoVersion:"go1.19.7", Compiler:"gc", Platform:"linux/amd64"}
```

Yet services remain unavailable and `kubelet` doesn't seem happy:

``` console
# systemctl status kubelet
○ kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: inactive (dead) since Tue 2024-03-05 22:34:03 CET; 22min ago
       Docs: https://kubernetes.io/docs/
   Main PID: 5986 (code=exited, status=0/SUCCESS)
        CPU: 5h 37min 33.871s

Mar 05 22:30:23 lexicon kubelet[5986]: I0305 22:30:23.982864    5986 pod_container_deletor.go:53] "DeleteContainer returned error" containerID={"Type":"containerd","ID":"4f70a3adfdba7962f514d636b31ca983549d95d7f9acfd81931296eddba50ff7"} err="failed to get container status \"4f70a3ad>
Mar 05 22:30:23 lexicon kubelet[5986]: I0305 22:30:23.982877    5986 scope.go:117] "RemoveContainer" containerID="3449079cc4940249b6798267ee61f4a39639c53a4701cb04b19b084546fb450e"
Mar 05 22:30:23 lexicon kubelet[5986]: E0305 22:30:23.983075    5986 remote_runtime.go:432] "ContainerStatus from runtime service failed" err="rpc error: code = NotFound desc = an error occurred when try to find container \"3449079cc4940249b6798267ee61f4a39639c53a4701cb04b19b084546f>
Mar 05 22:30:23 lexicon kubelet[5986]: I0305 22:30:23.983095    5986 pod_container_deletor.go:53] "DeleteContainer returned error" containerID={"Type":"containerd","ID":"3449079cc4940249b6798267ee61f4a39639c53a4701cb04b19b084546fb450e"} err="failed to get container status \"3449079c>
Mar 05 22:30:25 lexicon kubelet[5986]: I0305 22:30:25.669377    5986 kubelet_volumes.go:161] "Cleaned up orphaned pod volumes dir" podUID="681ab091-7025-4590-918c-106aa17b63cb" path="/var/lib/kubelet/pods/681ab091-7025-4590-918c-106aa17b63cb/volumes"
Mar 05 22:34:03 lexicon systemd[1]: Stopping kubelet: The Kubernetes Node Agent...
Mar 05 22:34:03 lexicon kubelet[5986]: I0305 22:34:03.171189    5986 dynamic_cafile_content.go:171] "Shutting down controller" name="client-ca-bundle::/etc/kubernetes/pki/ca.crt"
Mar 05 22:34:03 lexicon systemd[1]: kubelet.service: Deactivated successfully.
Mar 05 22:34:03 lexicon systemd[1]: Stopped kubelet: The Kubernetes Node Agent.
Mar 05 22:34:03 lexicon systemd[1]: kubelet.service: Consumed 5h 37min 33.871s CPU time.
```

Going back again to the [Single-node Kubernetes cluster on Ubuntu Server (lexicon)](https://stibbons1990.github.io/hex/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html)
and checking the current output from the first few `kubectl`, 
turns out version 1.28 is still running:

``` console
$ kubectl get nodes -o wide
NAME      STATUS     ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
lexicon   NotReady   control-plane   349d   v1.28.2   10.0.0.6      <none>        Ubuntu 22.04.4 LTS   5.15.0-97-generic   containerd://1.6.28
```

Deployment are all **not ready**:

``` console
$ kubectl get deployments -A
NAMESPACE                NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
atuin-server             atuin                                   0/1     1            0           184d
audiobookshelf           audiobookshelf                          0/1     1            0           4d12h
cert-manager             cert-manager                            0/1     1            0           324d
cert-manager             cert-manager-cainjector                 0/1     1            0           324d
cert-manager             cert-manager-webhook                    0/1     1            0           324d
code-server              code-server                             0/1     1            0           321d
default                  inteldeviceplugins-controller-manager   0/1     1            0           172d
ingress-nginx            ingress-nginx-controller                0/1     1            0           338d
kube-system              coredns                                 0/2     2            0           349d
kubernetes-dashboard     dashboard-metrics-scraper               0/1     1            0           338d
kubernetes-dashboard     kubernetes-dashboard                    0/1     1            0           338d
local-path-storage       local-path-provisioner                  0/1     1            0           324d
metallb-system           controller                              0/1     1            0           338d
metrics-server           metrics-server                          0/1     1            0           338d
node-feature-discovery   nfd-node-feature-discovery-master       0/1     1            0           172d
plexserver               plexserver                              0/1     1            0           156d
telegraf                 telegraf                                0/1     1            0           320d
```

At this point it seems necessary to manually restart `kubelet`:

``` console
# systemctl restart kubelet
# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Tue 2024-03-05 23:13:17 CET; 5s ago
       Docs: https://kubernetes.io/docs/
   Main PID: 3869533 (kubelet)
      Tasks: 13 (limit: 37926)
     Memory: 122.6M
        CPU: 701ms
     CGroup: /system.slice/kubelet.service
             └─3869533 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:/run/containerd/containerd.sock --pod-infra-container-image=registr>

Mar 05 23:13:21 lexicon kubelet[3869533]: I0305 23:13:21.200985 3869533 kubelet_volumes.go:160] "Cleaned up orphaned pod volumes dir" podUID=7f8e47b71a845b4eb634a44a94e20069 path="/var/lib/kubelet/pods/7f8e47b71a845b4eb634a44a94e20069/volumes"
Mar 05 23:13:21 lexicon kubelet[3869533]: I0305 23:13:21.201640 3869533 kubelet_volumes.go:160] "Cleaned up orphaned pod volumes dir" podUID=b087540859a3a8090a0d1bf55891aa52 path="/var/lib/kubelet/pods/b087540859a3a8090a0d1bf55891aa52/volumes"
Mar 05 23:13:21 lexicon kubelet[3869533]: W0305 23:13:21.333208 3869533 reflector.go:424] object-"kubernetes-dashboard"/"kube-root-ca.crt": failed to list *v1.ConfigMap: Get "https://10.0.0.6:6443/api/v1/namespaces/kubernetes-dashboard/configmaps?fieldSelector=metadata.name%3Dkube-r>
Mar 05 23:13:21 lexicon kubelet[3869533]: E0305 23:13:21.333302 3869533 reflector.go:140] object-"kubernetes-dashboard"/"kube-root-ca.crt": Failed to watch *v1.ConfigMap: failed to list *v1.ConfigMap: Get "https://10.0.0.6:6443/api/v1/namespaces/kubernetes-dashboard/configmaps?field>
Mar 05 23:13:21 lexicon kubelet[3869533]: W0305 23:13:21.533653 3869533 reflector.go:424] object-"default"/"webhook-server-cert": failed to list *v1.Secret: Get "https://10.0.0.6:6443/api/v1/namespaces/default/secrets?fieldSelector=metadata.name%3Dwebhook-server-cert&limit=500&resou>
Mar 05 23:13:21 lexicon kubelet[3869533]: E0305 23:13:21.533750 3869533 reflector.go:140] object-"default"/"webhook-server-cert": Failed to watch *v1.Secret: failed to list *v1.Secret: Get "https://10.0.0.6:6443/api/v1/namespaces/default/secrets?fieldSelector=metadata.name%3Dwebhook>
Mar 05 23:13:21 lexicon kubelet[3869533]: W0305 23:13:21.732794 3869533 reflector.go:424] object-"default"/"kube-root-ca.crt": failed to list *v1.ConfigMap: Get "https://10.0.0.6:6443/api/v1/namespaces/default/configmaps?fieldSelector=metadata.name%3Dkube-root-ca.crt&limit=500&resou>
Mar 05 23:13:21 lexicon kubelet[3869533]: E0305 23:13:21.732877 3869533 reflector.go:140] object-"default"/"kube-root-ca.crt": Failed to watch *v1.ConfigMap: failed to list *v1.ConfigMap: Get "https://10.0.0.6:6443/api/v1/namespaces/default/configmaps?fieldSelector=metadata.name%3Dk>
Mar 05 23:13:21 lexicon kubelet[3869533]: W0305 23:13:21.933227 3869533 reflector.go:424] object-"metrics-server"/"kube-root-ca.crt": failed to list *v1.ConfigMap: Get "https://10.0.0.6:6443/api/v1/namespaces/metrics-server/configmaps?fieldSelector=metadata.name%3Dkube-root-ca.crt&l>
Mar 05 23:13:21 lexicon kubelet[3869533]: E0305 23:13:21.933273 3869533 reflector.go:140] object-"metrics-server"/"kube-root-ca.crt": Failed to watch *v1.ConfigMap: failed to list *v1.ConfigMap: Get "https://10.0.0.6:6443/api/v1/namespaces/metrics-server/configmaps?fieldSelector=met>
```

Those logs entries look bad and yet everything seems to work now (and deployment are all ready):

``` console
$ kubectl get deployment -A 
NAMESPACE                NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
atuin-server             atuin                                   1/1     1            1           184d
audiobookshelf           audiobookshelf                          1/1     1            1           4d13h
cert-manager             cert-manager                            1/1     1            1           324d
cert-manager             cert-manager-cainjector                 1/1     1            1           324d
cert-manager             cert-manager-webhook                    1/1     1            1           324d
code-server              code-server                             1/1     1            1           321d
default                  inteldeviceplugins-controller-manager   1/1     1            1           172d
ingress-nginx            ingress-nginx-controller                1/1     1            1           338d
kube-system              coredns                                 2/2     2            2           349d
kubernetes-dashboard     dashboard-metrics-scraper               1/1     1            1           338d
kubernetes-dashboard     kubernetes-dashboard                    1/1     1            1           338d
local-path-storage       local-path-provisioner                  1/1     1            1           324d
metallb-system           controller                              1/1     1            1           338d
metrics-server           metrics-server                          1/1     1            1           338d
node-feature-discovery   nfd-node-feature-discovery-master       1/1     1            1           172d
plexserver               plexserver                              1/1     1            1           156d
telegraf                 telegraf                                1/1     1            1           320d
```
