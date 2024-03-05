---
title:  "Updating Kubernetes to the Community-Owned Package Repositories"
date:   2024-03-05 22:03:05 +0200
categories: linux kubernetes docker server ubuntu
---

Most days I update my little server when I log into my PC,
and today it gave quite an unexpected surprise:

```
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

Note that there is now a separate repository for each minor version,
so we need to first check which version is currently running:

```
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.2", GitCommit:"89a4ea3e1e4ddd7f7572286090359983e0387b2f", GitTreeState:"clean", BuildDate:"2023-09-13T09:34:32Z", GoVersion:"go1.20.8", Compiler:"gc", Platform:"linux/amd64"}
```

That we migrate to the **1.28** repository:

```
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
