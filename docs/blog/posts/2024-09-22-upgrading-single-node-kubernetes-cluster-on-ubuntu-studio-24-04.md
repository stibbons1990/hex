---
date: 2024-09-22
categories:
 - linux
 - ubuntu
 - server
 - kubernetes
 - docker
title: Upgrading single-node Kubernetes cluster on Ubuntu Studio 24.04
---

Last month I took a look at
[checking deployments before upgrading kubeadm clusters](../../../../2024/08/22/checking-deployments-before-upgrades.md)
and found results *mostly* reassuring.

<!-- more -->

As a practice run to upgrade more complex setups, lets upgrade
[the cluster running on the desktop PC](../../../../2024/05/12/single-node-kubernetes-cluster-on-ubuntu-studio-desktop-rapture.md),
which is only running a Plex Media Server (which recently become
unresponsive) and the
[PhotoPrism®](../../../../2024/08/24/self-hosted-photo-albums-with-photoprism.md) photo album (which
never worked well enough to be critical to me).

The cluster is running version 1.26 (which is quite old):

```
# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"26", GitVersion:"v1.26.15", GitCommit:"1649f592f1909b97aa3c2a0a8f968a3fd05a7b8b", GitTreeState:"clean", BuildDate:"2024-03-14T01:03:33Z", GoVersion:"go1.21.8", Compiler:"gc", Platform:"linux/amd64"}
```

Kubernetes clusters can only be upgraded one minor version at a
time (skipping minor version is not supported), so we start with
[upgrading a Kubernetes cluster created with kubeadm from version 1.26.x to version 1.27.x](https://v1-27.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).

Although the workloads are not critical, lets pretend they are and
[use `kubectl drain` to remove the node from service](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/#use-kubectl-drain-to-remove-a-node-from-service)
to ensure all pods are shut down gracefully:

```
# kubectl get nodes
NAME      STATUS   ROLES           AGE    VERSION
rapture   Ready    control-plane   132d   v1.26.15

# kubectl drain --ignore-daemonsets rapture
cordoned
error: unable to drain node "rapture" due to error:cannot delete Pods with local storage (use --delete-emptydir-data to override): kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-s77ss, kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-9dxc2, continuing command...
There are pending nodes to be drained:
 rapture
cannot delete Pods with local storage (use --delete-emptydir-data to override): kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-s77ss, kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-9dxc2
```

Turns out this clustomers needs the `--delete-emptydir-data` flag
because *there are pods using emptyDir* and *local data that will
be deleted when the node is drained*.

```
# kubectl drain --ignore-daemonsets rapture --delete-emptydir-data
node/rapture already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-bbhn8, kube-system/kube-proxy-tszr7, metallb-system/speaker-58krr
evicting pod ingress-nginx/ingress-nginx-admission-create-nfm7b
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-jshnd
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-5ncn8
evicting pod kube-system/coredns-787d4945fb-k8m6b
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-bhzkt
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-78t4g
evicting pod ingress-nginx/ingress-nginx-admission-patch-8x6r7
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-qrj7s
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-4lwwm
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-5pgkv
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-b2n6c
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-4sswx
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-6b5wz
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-6v4ds
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-67p2t
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-7r4xz
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-4nnmr
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-nhw62
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-gzvjt
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-dbpwt
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-xkvg8
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-qtljx
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-5dj66
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-hzzsl
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-9ndhr
evicting pod metallb-system/controller-759b6c5bb8-6sp2m
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-nthjr
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-mvs2g
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-bkppp
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-krkdp
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-9vz7w
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-s77ss
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-l5pm9
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-bctwj
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-brzmr
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-mktgp
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-t64ks
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-p6l6x
evicting pod default/command-demo
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-twnjf
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-cbfb4
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-dmlzv
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-9dxc2
evicting pod metallb-system/controller-759b6c5bb8-zfxwn
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-v22hn
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-ntftm
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-zdv25
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-7g5s8
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-9kwrh
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-w5pzx
evicting pod photoprism/photoprism-0
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-nv9pk
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-n6vpl
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-hxwmf
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-qgwjz
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-djsjh
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-9qg2h
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-dqrn8
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-r5cng
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-42q9b
evicting pod metallb-system/controller-759b6c5bb8-57nb6
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-b22xz
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-rjrjb
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-f2kfh
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-952v2
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-b6nwl
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-dfc2k
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-bs26f
evicting pod metallb-system/controller-759b6c5bb8-9zbn4
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-pc2f7
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-4tgjq
evicting pod metallb-system/controller-759b6c5bb8-b2pch
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-c82nc
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-qbbmd
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-bjw5s
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-bkt7v
evicting pod metallb-system/controller-759b6c5bb8-bl2tn
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-qz758
evicting pod metallb-system/controller-759b6c5bb8-cbsq4
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-x5trq
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-bm6bw
evicting pod metallb-system/controller-759b6c5bb8-cvkmj
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-ss2sq
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-b8lzj
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-s5sr7
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-jfjsx
evicting pod metallb-system/controller-759b6c5bb8-f6zjm
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-fl49q
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-8h9rw
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-s9tcr
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-cqrhs
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-pxb5m
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-pzgln
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-pls88
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-t27j4
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-pzmbn
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-prw6v
evicting pod metallb-system/controller-759b6c5bb8-g9g2x
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-jn4bt
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-8gzvj
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-zrrb8
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-krrtr
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-hkxzh
evicting pod metallb-system/controller-759b6c5bb8-ht7tz
evicting pod metallb-system/controller-759b6c5bb8-nr92c
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-n2gn8
evicting pod metallb-system/controller-759b6c5bb8-ktq7h
evicting pod metallb-system/controller-759b6c5bb8-xx5bb
evicting pod metallb-system/controller-759b6c5bb8-hx4dl
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-lbzrz
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-n8cxk
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-m8mb8
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-vd88s
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-7bjzs
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-45qcw
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-kbhtg
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-fx98h
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-hjwcl
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-ztfwd
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-v8lkd
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-kt46k
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-mbkmb
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-fscmh
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-6n2zd
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-wbrk4
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-x462f
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-26hxx
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-zs228
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-9htbw
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-hxnnz
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-74flg
evicting pod metallb-system/controller-759b6c5bb8-jvpbg
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-vkfxn
evicting pod metallb-system/controller-759b6c5bb8-2gpk2
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-dmws2
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-xr74p
evicting pod metallb-system/controller-759b6c5bb8-hxhzc
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-wlzfq
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-wgtll
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-qpp5w
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-hmjdg
evicting pod ingress-nginx/ingress-nginx-controller-7c7754d4b6-mgkdz
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-v66mn
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-nj2l7
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-7z57g
evicting pod kube-system/coredns-787d4945fb-sqfcw
evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-b94tb
evicting pod plexserver/plexserver-7cb77ddc8-d6bqg
pod/ingress-nginx-controller-7c7754d4b6-x5trq evicted
pod/dashboard-metrics-scraper-7bc864c59-bhzkt evicted
pod/kubernetes-dashboard-6c7ccbcf87-qz758 evicted
pod/controller-759b6c5bb8-zfxwn evicted
pod/kubernetes-dashboard-6c7ccbcf87-v22hn evicted
pod/kubernetes-dashboard-6c7ccbcf87-hzzsl evicted
pod/kubernetes-dashboard-6c7ccbcf87-l5pm9 evicted
pod/controller-759b6c5bb8-cbsq4 evicted
pod/dashboard-metrics-scraper-7bc864c59-67p2t evicted
pod/kubernetes-dashboard-6c7ccbcf87-bm6bw evicted
pod/kubernetes-dashboard-6c7ccbcf87-nthjr evicted
pod/kubernetes-dashboard-6c7ccbcf87-bctwj evicted
I0922 09:37:13.980459 1271042 request.go:690] Waited for 1.000499779s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.2:6443/api/v1/namespaces/metallb-system/pods/controller-759b6c5bb8-cvkmj/eviction
pod/controller-759b6c5bb8-cvkmj evicted
pod/kubernetes-dashboard-6c7ccbcf87-9dxc2 evicted
pod/dashboard-metrics-scraper-7bc864c59-78t4g evicted
pod/dashboard-metrics-scraper-7bc864c59-ntftm evicted
pod/dashboard-metrics-scraper-7bc864c59-6b5wz evicted
pod/dashboard-metrics-scraper-7bc864c59-ss2sq evicted
pod/kubernetes-dashboard-6c7ccbcf87-brzmr evicted
pod/ingress-nginx-controller-7c7754d4b6-xkvg8 evicted
pod/kubernetes-dashboard-6c7ccbcf87-zdv25 evicted
pod/kubernetes-dashboard-6c7ccbcf87-b8lzj evicted
pod/ingress-nginx-admission-patch-8x6r7 evicted
pod/dashboard-metrics-scraper-7bc864c59-qtljx evicted
pod/kubernetes-dashboard-6c7ccbcf87-s5sr7 evicted
pod/dashboard-metrics-scraper-7bc864c59-mktgp evicted
pod/kubernetes-dashboard-6c7ccbcf87-nhw62 evicted
pod/kubernetes-dashboard-6c7ccbcf87-jfjsx evicted
pod/ingress-nginx-controller-7c7754d4b6-qrj7s evicted
pod/kubernetes-dashboard-6c7ccbcf87-gzvjt evicted
pod/dashboard-metrics-scraper-7bc864c59-t64ks evicted
pod/kubernetes-dashboard-6c7ccbcf87-dbpwt evicted
pod/dashboard-metrics-scraper-7bc864c59-5dj66 evicted
pod/dashboard-metrics-scraper-7bc864c59-4lwwm evicted
pod/ingress-nginx-admission-create-nfm7b evicted
pod/controller-759b6c5bb8-f6zjm evicted
pod/kubernetes-dashboard-6c7ccbcf87-cbfb4 evicted
pod/dashboard-metrics-scraper-7bc864c59-p6l6x evicted
pod/dashboard-metrics-scraper-7bc864c59-5pgkv evicted
pod/coredns-787d4945fb-k8m6b evicted
pod/dashboard-metrics-scraper-7bc864c59-fl49q evicted
pod/kubernetes-dashboard-6c7ccbcf87-djsjh evicted
pod/ingress-nginx-controller-7c7754d4b6-8h9rw evicted
pod/kubernetes-dashboard-6c7ccbcf87-9qg2h evicted
pod/command-demo evicted
pod/kubernetes-dashboard-6c7ccbcf87-s9tcr evicted
pod/dashboard-metrics-scraper-7bc864c59-dmlzv evicted
pod/dashboard-metrics-scraper-7bc864c59-hxwmf evicted
pod/kubernetes-dashboard-6c7ccbcf87-twnjf evicted
pod/kubernetes-dashboard-6c7ccbcf87-7r4xz evicted
pod/dashboard-metrics-scraper-7bc864c59-qgwjz evicted
pod/kubernetes-dashboard-6c7ccbcf87-w5pzx evicted
pod/kubernetes-dashboard-6c7ccbcf87-9kwrh evicted
pod/dashboard-metrics-scraper-7bc864c59-nv9pk evicted
pod/photoprism-0 evicted
pod/dashboard-metrics-scraper-7bc864c59-4sswx evicted
pod/dashboard-metrics-scraper-7bc864c59-bkppp evicted
pod/kubernetes-dashboard-6c7ccbcf87-7g5s8 evicted
I0922 09:37:24.142398 1271042 request.go:690] Waited for 1.349459923s due to client-side throttling, not priority and fairness, request: GET:https://10.0.0.2:6443/api/v1/namespaces/kubernetes-dashboard/pods/dashboard-metrics-scraper-7bc864c59-6v4ds
pod/dashboard-metrics-scraper-7bc864c59-6v4ds evicted
pod/dashboard-metrics-scraper-7bc864c59-42q9b evicted
pod/kubernetes-dashboard-6c7ccbcf87-9ndhr evicted
pod/dashboard-metrics-scraper-7bc864c59-dqrn8 evicted
pod/dashboard-metrics-scraper-7bc864c59-9vz7w evicted
pod/ingress-nginx-controller-7c7754d4b6-mvs2g evicted
pod/kubernetes-dashboard-6c7ccbcf87-b22xz evicted
pod/kubernetes-dashboard-6c7ccbcf87-pc2f7 evicted
pod/dashboard-metrics-scraper-7bc864c59-5ncn8 evicted
pod/kubernetes-dashboard-6c7ccbcf87-qbbmd evicted
pod/kubernetes-dashboard-6c7ccbcf87-bkt7v evicted
pod/dashboard-metrics-scraper-7bc864c59-rjrjb evicted
pod/controller-759b6c5bb8-bl2tn evicted
pod/kubernetes-dashboard-6c7ccbcf87-krkdp evicted
pod/ingress-nginx-controller-7c7754d4b6-4tgjq evicted
pod/dashboard-metrics-scraper-7bc864c59-c82nc evicted
pod/dashboard-metrics-scraper-7bc864c59-f2kfh evicted
pod/kubernetes-dashboard-6c7ccbcf87-bjw5s evicted
pod/ingress-nginx-controller-7c7754d4b6-dfc2k evicted
pod/controller-759b6c5bb8-57nb6 evicted
pod/controller-759b6c5bb8-b2pch evicted
pod/controller-759b6c5bb8-9zbn4 evicted
pod/kubernetes-dashboard-6c7ccbcf87-b6nwl evicted
pod/dashboard-metrics-scraper-7bc864c59-s77ss evicted
pod/dashboard-metrics-scraper-7bc864c59-bs26f evicted
pod/dashboard-metrics-scraper-7bc864c59-952v2 evicted
pod/dashboard-metrics-scraper-7bc864c59-n6vpl evicted
pod/ingress-nginx-controller-7c7754d4b6-4nnmr evicted
pod/dashboard-metrics-scraper-7bc864c59-cqrhs evicted
pod/dashboard-metrics-scraper-7bc864c59-r5cng evicted
pod/dashboard-metrics-scraper-7bc864c59-pxb5m evicted
pod/dashboard-metrics-scraper-7bc864c59-pzgln evicted
pod/dashboard-metrics-scraper-7bc864c59-pls88 evicted
pod/kubernetes-dashboard-6c7ccbcf87-t27j4 evicted
pod/dashboard-metrics-scraper-7bc864c59-pzmbn evicted
pod/dashboard-metrics-scraper-7bc864c59-prw6v evicted
pod/controller-759b6c5bb8-g9g2x evicted
pod/dashboard-metrics-scraper-7bc864c59-b2n6c evicted
pod/dashboard-metrics-scraper-7bc864c59-jn4bt evicted
pod/kubernetes-dashboard-6c7ccbcf87-8gzvj evicted
pod/dashboard-metrics-scraper-7bc864c59-zrrb8 evicted
pod/dashboard-metrics-scraper-7bc864c59-krrtr evicted
pod/dashboard-metrics-scraper-7bc864c59-hkxzh evicted
pod/controller-759b6c5bb8-ht7tz evicted
pod/controller-759b6c5bb8-nr92c evicted
pod/kubernetes-dashboard-6c7ccbcf87-n2gn8 evicted
pod/controller-759b6c5bb8-ktq7h evicted
pod/controller-759b6c5bb8-xx5bb evicted
I0922 09:37:34.180242 1271042 request.go:690] Waited for 21.199712674s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.2:6443/api/v1/namespaces/kubernetes-dashboard/pods/dashboard-metrics-scraper-7bc864c59-kbhtg/eviction
pod/controller-759b6c5bb8-6sp2m evicted
pod/controller-759b6c5bb8-hx4dl evicted
pod/kubernetes-dashboard-6c7ccbcf87-lbzrz evicted
pod/kubernetes-dashboard-6c7ccbcf87-n8cxk evicted
pod/kubernetes-dashboard-6c7ccbcf87-m8mb8 evicted
pod/dashboard-metrics-scraper-7bc864c59-vd88s evicted
pod/dashboard-metrics-scraper-7bc864c59-7bjzs evicted
pod/kubernetes-dashboard-6c7ccbcf87-45qcw evicted
pod/dashboard-metrics-scraper-7bc864c59-kbhtg evicted
pod/kubernetes-dashboard-6c7ccbcf87-fx98h evicted
pod/dashboard-metrics-scraper-7bc864c59-hjwcl evicted
pod/dashboard-metrics-scraper-7bc864c59-ztfwd evicted
pod/dashboard-metrics-scraper-7bc864c59-v8lkd evicted
pod/dashboard-metrics-scraper-7bc864c59-kt46k evicted
pod/dashboard-metrics-scraper-7bc864c59-mbkmb evicted
pod/kubernetes-dashboard-6c7ccbcf87-fscmh evicted
pod/kubernetes-dashboard-6c7ccbcf87-6n2zd evicted
pod/dashboard-metrics-scraper-7bc864c59-wbrk4 evicted
pod/kubernetes-dashboard-6c7ccbcf87-x462f evicted
pod/kubernetes-dashboard-6c7ccbcf87-26hxx evicted
pod/kubernetes-dashboard-6c7ccbcf87-zs228 evicted
pod/kubernetes-dashboard-6c7ccbcf87-9htbw evicted
pod/dashboard-metrics-scraper-7bc864c59-hxnnz evicted
pod/kubernetes-dashboard-6c7ccbcf87-74flg evicted
pod/controller-759b6c5bb8-jvpbg evicted
pod/kubernetes-dashboard-6c7ccbcf87-vkfxn evicted
pod/controller-759b6c5bb8-2gpk2 evicted
pod/dashboard-metrics-scraper-7bc864c59-dmws2 evicted
pod/kubernetes-dashboard-6c7ccbcf87-xr74p evicted
pod/controller-759b6c5bb8-hxhzc evicted
pod/ingress-nginx-controller-7c7754d4b6-jshnd evicted
pod/kubernetes-dashboard-6c7ccbcf87-wlzfq evicted
pod/dashboard-metrics-scraper-7bc864c59-wgtll evicted
pod/dashboard-metrics-scraper-7bc864c59-qpp5w evicted
pod/dashboard-metrics-scraper-7bc864c59-hmjdg evicted
pod/ingress-nginx-controller-7c7754d4b6-mgkdz evicted
pod/dashboard-metrics-scraper-7bc864c59-v66mn evicted
pod/dashboard-metrics-scraper-7bc864c59-nj2l7 evicted
pod/dashboard-metrics-scraper-7bc864c59-7z57g evicted
pod/kubernetes-dashboard-6c7ccbcf87-b94tb evicted
pod/plexserver-7cb77ddc8-d6bqg evicted
pod/coredns-787d4945fb-sqfcw evicted
node/rapture drained
```

**Note:** local storage defined as `PersistentVolume` is **not**
deleted:

```
# du -sh /home/k8s/*
920M    /home/k8s/audiobookshelf
17G     /home/k8s/photoprism
7.6G    /home/k8s/plexmediaserver
```

At this point the Kubernetes cluster is running everything **but**
those pods that are not part of Kubernetes:

```
# kubectl get deployments -A
NAMESPACE              NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
ingress-nginx          ingress-nginx-controller    0/1     1            0           132d
kube-system            coredns                     0/2     2            0           132d
kubernetes-dashboard   dashboard-metrics-scraper   0/1     1            0           132d
kubernetes-dashboard   kubernetes-dashboard        0/1     1            0           132d
metallb-system         controller                  0/1     1            0           132d
plexserver             plexserver                  0/1     1            0           132d

# kubectl get pods -A
NAMESPACE              NAME                                        READY   STATUS    RESTARTS        AGE
ingress-nginx          ingress-nginx-controller-7c7754d4b6-w4w5g   0/1     Pending   0               9m15s
kube-flannel           kube-flannel-ds-bbhn8                       1/1     Running   120 (27h ago)   132d
kube-system            coredns-787d4945fb-8xkqt                    0/1     Pending   0               9m3s
kube-system            coredns-787d4945fb-kjrlx                    0/1     Pending   0               9m30s
kube-system            etcd-rapture                                1/1     Running   89 (27h ago)    132d
kube-system            kube-apiserver-rapture                      1/1     Running   89 (27h ago)    132d
kube-system            kube-controller-manager-rapture             1/1     Running   89 (27h ago)    132d
kube-system            kube-proxy-tszr7                            1/1     Running   89 (27h ago)    132d
kube-system            kube-scheduler-rapture                      1/1     Running   89 (27h ago)    132d
kubernetes-dashboard   dashboard-metrics-scraper-7bc864c59-4lw4r   0/1     Pending   0               9m16s
kubernetes-dashboard   kubernetes-dashboard-6c7ccbcf87-mg8j6       0/1     Pending   0               9m30s
metallb-system         controller-759b6c5bb8-d4lwr                 0/1     Pending   0               9m25s
metallb-system         speaker-58krr                               1/1     Running   50 (27h ago)    76d
photoprism             photoprism-0                                0/1     Pending   0               9m21s
plexserver             plexserver-7cb77ddc8-rn2p6                  0/1     Pending   0               9m3s
```

At this point it would be safe to stop all the services; but this
*is not necessary* (and may get in the way of upgrading):

```
# systemctl stop kubelet
# systemctl stop containerd.service
# systemctl stop docker.service
# systemctl stop docker.socket
```

**Instead**, the next step is to
[determine which version to upgrade to](https://v1-27.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#determine-which-version-to-upgrade-to)
by updating the minor version in the repository configuration and
then finding the latest patch version:

```
# vi /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /

# apt update
# apt-cache madison kubeadm
   kubeadm | 1.27.16-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.15-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.14-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.13-2.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.12-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.11-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.10-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.0-2.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
```

The latest patch version is 1.27.**16** and that is the one to
[upgrade control plane nodes](https://v1-27.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes) to:

```
# apt-mark unhold kubeadm && \
  apt-get update && apt-get install -y kubeadm='1.27.16-*' && \
  apt-mark hold kubeadm
kubeadm was already not on hold.
...
Selected version '1.27.16-1.1' (isv:kubernetes:core:stable:v1.27:pkgs.k8s.io [amd64]) for 'kubeadm'
The following packages will be upgraded:
  kubeadm
1 upgraded, 0 newly installed, 0 to remove and 5 not upgraded.
Need to get 10.0 MB of archives.
After this operation, 848 kB of additional disk space will be used.
Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  kubeadm 1.27.16-1.1 [10.0 MB]
Fetched 10.0 MB in 1s (15.8 MB/s)
(Reading database ... 645263 files and directories currently installed.)
Preparing to unpack .../kubeadm_1.27.16-1.1_amd64.deb ...
Unpacking kubeadm (1.27.16-1.1) over (1.26.15-1.1) ...
Setting up kubeadm (1.27.16-1.1) ...
kubeadm set on hold.

# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.16", GitCommit:"cbb86e0d7f4a049666fac0551e8b02ef3d6c3d9a", GitTreeState:"clean", BuildDate:"2024-07-17T01:52:04Z", GoVersion:"go1.22.5", Compiler:"gc", Platform:"linux/amd64"}

# kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.26.15
[upgrade/versions] kubeadm version: v1.27.16
I0922 10:02:36.788943 1866090 version.go:256] remote version is much newer: v1.31.1; falling back to: stable-1.27
[upgrade/versions] Target version: v1.27.16
[upgrade/versions] Latest version in the v1.26 series: v1.26.15

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT        TARGET
kubelet     1 x v1.26.15   v1.27.16

Upgrade to the latest stable version:

COMPONENT                 CURRENT    TARGET
kube-apiserver            v1.26.15   v1.27.16
kube-controller-manager   v1.26.15   v1.27.16
kube-scheduler            v1.26.15   v1.27.16
kube-proxy                v1.26.15   v1.27.16
CoreDNS                   v1.9.3     v1.10.1
etcd                      3.5.10-0   3.5.12-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.27.16

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________


# kubeadm upgrade apply v1.27.16
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.27.16"
[upgrade/versions] Cluster version: v1.26.15
[upgrade/versions] kubeadm version: v1.27.16
[upgrade] Are you sure you want to proceed? [y/N]: y
[upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
[upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
[upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
W0922 10:04:12.442139 1888818 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.27.16" (timeout: 5m0s)...
[upgrade/etcd] Upgrading to TLS for etcd
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-09-22-10-04-17/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests446600773"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-09-22-10-04-17/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-09-22-10-04-17/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-09-22-10-04-17/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config4276155515/config.yaml
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.27.16". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.16", GitCommit:"cbb86e0d7f4a049666fac0551e8b02ef3d6c3d9a", GitTreeState:"clean", BuildDate:"2024-07-17T01:52:04Z", GoVersion:"go1.22.5", Compiler:"gc", Platform:"linux/amd64"}
```

Now that the control plane is updated, proceed to
[upgrade kubelet and kubectl](https://v1-27.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl)

```
# apt-mark unhold kubelet kubectl && \
  apt-get update && apt-get install -y kubelet='1.27.16-*' kubectl='1.27.16-*' && \
  apt-mark hold kubelet kubectl
kubelet was already not on hold.
kubectl was already not on hold.
...
Preparing to unpack .../kubectl_1.27.16-1.1_amd64.deb ...
Unpacking kubectl (1.27.16-1.1) over (1.26.15-1.1) ...
Preparing to unpack .../kubelet_1.27.16-1.1_amd64.deb ...
Unpacking kubelet (1.27.16-1.1) over (1.26.15-1.1) ...
dpkg: warning: unable to delete old directory '/etc/sysconfig': Directory not empty
Setting up kubectl (1.27.16-1.1) ...
Setting up kubelet (1.27.16-1.1) ...
kubelet set on hold.
kubectl set on hold.

# systemctl daemon-reload
# systemctl restart kubelet

# kubectl  version --output=yaml
clientVersion:
  buildDate: "2024-07-17T01:53:56Z"
  compiler: gc
  gitCommit: cbb86e0d7f4a049666fac0551e8b02ef3d6c3d9a
  gitTreeState: clean
  gitVersion: v1.27.16
  goVersion: go1.22.5
  major: "1"
  minor: "27"
  platform: linux/amd64
kustomizeVersion: v5.0.1
serverVersion:
  buildDate: "2024-07-17T01:44:26Z"
  compiler: gc
  gitCommit: cbb86e0d7f4a049666fac0551e8b02ef3d6c3d9a
  gitTreeState: clean
  gitVersion: v1.27.16
  goVersion: go1.22.5
  major: "1"
  minor: "27"
  platform: linux/amd64
```

Finally, bring the node back online by marking it schedulable:

```
# kubectl uncordon rapture
node/rapture uncordoned

# kubectl get pods -A
NAMESPACE              NAME                                        READY   STATUS        RESTARTS        AGE
ingress-nginx          ingress-nginx-controller-7c7754d4b6-w4w5g   0/1     Running       0               70m
kube-flannel           kube-flannel-ds-bbhn8                       1/1     Running       120 (28h ago)   132d
kube-system            coredns-5d78c9869d-9xbh4                    1/1     Running       0               41m
kube-system            coredns-5d78c9869d-w8954                    1/1     Running       0               41m
kube-system            coredns-787d4945fb-8xkqt                    1/1     Terminating   0               70m
kube-system            etcd-rapture                                1/1     Running       2 (72s ago)     73s
kube-system            kube-apiserver-rapture                      1/1     Running       2 (72s ago)     73s
kube-system            kube-controller-manager-rapture             1/1     Running       2 (72s ago)     73s
kube-system            kube-proxy-nzmwz                            1/1     Running       0               40m
kube-system            kube-scheduler-rapture                      1/1     Running       1 (72s ago)     73s
kubernetes-dashboard   dashboard-metrics-scraper-7bc864c59-4lw4r   1/1     Running       0               70m
kubernetes-dashboard   kubernetes-dashboard-6c7ccbcf87-mg8j6       1/1     Running       0               70m
metallb-system         controller-759b6c5bb8-d4lwr                 0/1     Running       0               70m
metallb-system         speaker-58krr                               1/1     Running       50 (28h ago)    76d
photoprism             photoprism-0                                0/1     Running       0               70m
plexserver             plexserver-7cb77ddc8-rn2p6                  1/1     Running       0               70m
```

After a few minutes, the Plex Media Server and the PhotoPrism®
services are up and running again.

In the end the process seems to have gone pretty smoothly. With
this out of the way, and given that
[deployments passed checks for deprecations](../../../../2024/08/22/checking-deployments-before-upgrades.md),
the next step is to repeat this process for every minor version
up to the latest.
