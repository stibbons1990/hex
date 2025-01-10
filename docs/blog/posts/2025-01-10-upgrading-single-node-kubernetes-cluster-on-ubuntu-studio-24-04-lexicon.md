---
date: 2025-01-10
categories:
 - linux
 - ubuntu
 - server
 - kubernetes
 - docker
title: Upgrading single-node Kubernetes cluster on Ubuntu Studio 24.04 (lexicon)
---

[Upgrading the single-node kubernetes cluster on rapture](./2024-09-22-upgrading-single-node-kubernetes-cluster-on-ubuntu-studio-24-04.md)
went smoothly, so it's time to repeat the process on `lexicon`.

<!-- more -->

### Upgrade to 1.27

``` console
# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"26", GitVersion:"v1.26.15", GitCommit:"1649f592f1909b97aa3c2a0a8f968a3fd05a7b8b", GitTreeState:"clean", BuildDate:"2024-03-14T01:03:33Z", GoVersion:"go1.21.8", Compiler:"gc", Platform:"linux/amd64"}

# kubectl get nodes
NAME      STATUS   ROLES           AGE    VERSION
lexicon   Ready    control-plane   659d   v1.26.15
```

Again, the workloads are not critical, lets pretend they are and
[use `kubectl drain` to remove the node from service](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/#use-kubectl-drain-to-remove-a-node-from-service)
to ensure all pods are shut down gracefully:

``` console
# kubectl drain --ignore-daemonsets lexicon
node/lexicon cordoned
error: unable to drain node "lexicon" due to error:cannot delete Pods with local storage (use --delete-emptydir-data to override): kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-p27cg, kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-vwxnt, metrics-server/metrics-server-74c749979-wd278, continuing command...
There are pending nodes to be drained:
 lexicon
cannot delete Pods with local storage (use --delete-emptydir-data to override): kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-p27cg, kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-vwxnt, metrics-server/metrics-server-74c749979-wd278
```

This warning can be ignored safely, those are literally empty
directories which at most have a little JSON file:

``` console
# kubectl -n kubernetes-dashboard describe pod kubernetes-dashboard | grep containerd
    Container ID:  containerd://32baf2a5a439f9623a08fc6127a3d605140b09b46c40c8d535685daf5127bbb0

# find /home/lib/ -type d | grep 32baf2a5a439f9623a08fc6127a3d605140b09b46c40c8d535685daf5127bbb0
/home/lib/containerd/io.containerd.runtime.v2.task/k8s.io/32baf2a5a439f9623a08fc6127a3d605140b09b46c40c8d535685daf5127bbb0
/home/lib/containerd/io.containerd.grpc.v1.cri/containers/32baf2a5a439f9623a08fc6127a3d605140b09b46c40c8d535685daf5127bbb0

# ls -l /home/lib/containerd/io.containerd.runtime.v2.task/k8s.io/26e0473de095cd0d4cf505eb6b7938764349c7c5257eaf8540136d5037e3d69e /home/lib/containerd/io.containerd.grpc.v1.cri/containers/26e0473de095cd0d4cf505eb6b7938764349c7c5257eaf8540136d5037e3d69e
/home/lib/containerd/io.containerd.grpc.v1.cri/containers/26e0473de095cd0d4cf505eb6b7938764349c7c5257eaf8540136d5037e3d69e:
total 4
-rw------- 1 root root 225 Jan 10 06:08 status

/home/lib/containerd/io.containerd.runtime.v2.task/k8s.io/26e0473de095cd0d4cf505eb6b7938764349c7c5257eaf8540136d5037e3d69e:
total 0

# cat /home/lib/containerd/io.containerd.grpc.v1.cri/containers/32baf2a5a439f9623a08fc6127a3d605140b09b46c40c8d535685daf5127bbb0/status | jq -r .
{
  "Version": "v1",
  "Pid": 29784,
  "CreatedAt": 1736485731044226300,
  "StartedAt": 1736485731100578000,
  "FinishedAt": 0,
  "ExitCode": 0,
  "Reason": "",
  "Message": "",
  "Resources": {
    "linux": {
      "cpu_period": 100000,
      "cpu_shares": 2,
      "oom_score_adj": 1000
    }
  }
}
```

Thus, lets drain the code:

??? terminal "`# kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon`"

    ``` console
    # kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon
    node/lexicon already cordoned
    Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-nrrg6, kube-system/kube-proxy-qlt8c, metallb-system/speaker-t8f7k, monitoring/telegraf-ztg2t, node-feature-discovery/nfd-node-feature-discovery-worker-xmzwk
    evicting pod atuin-server/atuin-cdc5879b8-8kl24
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-jhhhc
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-cb9bh
    evicting pod cert-manager/cert-manager-64f9f45d6f-qx4hs
    evicting pod cert-manager/cert-manager-webhook-d4f4545d7-tf4l6
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-4gg7d
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-22z7l
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-2n2dz
    evicting pod code-server/code-server-5768c9d997-cr858
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-2xr55
    evicting pod default/inteldeviceplugins-controller-manager-7994555cdb-gjmfm
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-48vvc
    evicting pod firefly-iii/firefly-iii-5bd9bb7c4-vxv87
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-7qsnq
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-5v9tk
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-6jpm7
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-4lr52
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-6jvv4
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-4n488
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-6ldtx
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-4nr7s
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-m7gcs
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-jq85x
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-5st6w
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-4wphp
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-76rlb
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-hhv8z
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-m8q2t
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-lrqfc
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-8mrll
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-hlxqp
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-928zt
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-hpfx4
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-bbpm2
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-jf458
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-bncmb
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-jgk98
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-c68ws
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-c9khh
    evicting pod unifi/unifi-6678ccc684-7g9qb
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-728m7
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-7m8b6
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-mngbg
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-cdqq9
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-k7rlx
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-nntnl
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-kq25w
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-nptbm
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-ksqm2
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-ntrgw
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-p9gsv
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-pftg7
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-pnr9n
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-pvrt5
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-qck44
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-qfhzv
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-qxgfc
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-qzlfw
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-r7v2f
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-rltww
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-rpfnc
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-rx2kb
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-rx92b
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-s4ggm
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-sbhrt
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-sg6nw
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-sj4vv
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-sksd9
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-t2b58
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-t9fnn
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-tn62g
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-tvcdg
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-hd72r
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-jdvng
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-tzr6h
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-jmg44
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-vvl94
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-vlqdn
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-wv8qh
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-jmsfn
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-x6v5x
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-xf5qv
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-jtqtc
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-k8xvt
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-zcwd5
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-xh6s5
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-zp8fd
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-khdf4
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-zr662
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-kprpk
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-zrj82
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-lhffx
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-zttt5
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-lxxqk
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-mdvx4
    evicting pod firefly-iii/firefly-iii-mysql-68f59d48f-k5mpw
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-mp7k9
    evicting pod homebox/homebox-7fb9d44d48-wxj6q
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-mptfs
    evicting pod ingress-nginx/ingress-nginx-admission-create-7zfzs
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-nqnvc
    evicting pod ingress-nginx/ingress-nginx-admission-patch-4tvvk
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-pk8f4
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-244q5
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-qgb8v
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-24z6b
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-r2t6n
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-2brp6
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-r9kjw
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-2g7t8
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-2g2c2
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-2k6pl
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-rnz84
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-2mtgv
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-rt5lm
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-48gnx
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-s5w6f
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-4k4vc
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-s989f
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-4rktx
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-snhd6
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-smqsc
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-srhvn
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-4sldn
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-vdznd
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-4wcq2
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-vftsh
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-4wq8x
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-vsxpk
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-4ztpd
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-vxbpz
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-5m6w5
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-vz69m
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-5rnzx
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-wg556
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-66dpd
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-x8sw4
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-6gdb7
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-xg4bx
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-6tf9z
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-xmfkv
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-7h2qx
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-xndqz
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-87lgg
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-826lj
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-8p9kb
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-xtm2h
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-8rs7g
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-z5nr6
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-zp4tr
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-9lljr
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-zqd8d
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-9lmrg
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-zsfpv
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-9mfzf
    evicting pod ingress-nginx/ingress-nginx-controller-746f764ff4-x2xc2
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-9t8dd
    evicting pod kavita/kavita-766577cf45-dz9sl
    evicting pod kube-system/coredns-787d4945fb-67z8g
    evicting pod komga/komga-549b6dd9b-4zvb5
    evicting pod kube-system/coredns-787d4945fb-gsx6h
    evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-p27cg
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-bqk9h
    evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-vwxnt
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-bqw8m
    evicting pod local-path-storage/local-path-provisioner-8bc8875b-lbjh8
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-c77rw
    evicting pod metallb-system/controller-68bf958bf9-79mpk
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-c7dwq
    evicting pod metrics-server/metrics-server-74c749979-wd278
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-cmvt2
    evicting pod minecraft-server/minecraft-server-bd48db4bd-wjc22
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-ddf2h
    evicting pod monitoring/grafana-7647f97d64-k8h5m
    evicting pod monitoring/influxdb-57bddc4dc9-2z4v8
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-dksm6
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-dkjmb
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-f28xf
    evicting pod navidrome/navidrome-5f76b6ddf5-gcft4
    evicting pod node-feature-discovery/nfd-node-feature-discovery-master-5f56c75d-xjkb8
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-f2k7w
    evicting pod plexserver/plexserver-5dbdfc898b-9v2s8
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-f5pft
    evicting pod unifi/mongo-d55fbb7f5-pvdfr
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-ghjtz
    evicting pod ingress-nginx/ingress-nginx-controller-6b58ffdc97-gs2w6
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-ckx84
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-dgghp
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-dzlgj
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-f75kw
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-f8nm6
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-fphdc
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-fv6zx
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-fvfwr
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-fzq24
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-g28xp
    evicting pod firefly-iii/firefly-iii-7bfd57ff49-h97wd
    evicting pod cert-manager/cert-manager-cainjector-56bbdd5c47-ltjgx
    evicting pod audiobookshelf/audiobookshelf-987b955fb-64gxm
    pod/firefly-iii-7bfd57ff49-ksqm2 evicted
    pod/firefly-iii-7bfd57ff49-kq25w evicted
    pod/firefly-iii-7bfd57ff49-nntnl evicted
    pod/firefly-iii-7bfd57ff49-cdqq9 evicted
    pod/firefly-iii-7bfd57ff49-728m7 evicted
    pod/firefly-iii-7bfd57ff49-ntrgw evicted
    pod/firefly-iii-7bfd57ff49-mngbg evicted
    pod/firefly-iii-7bfd57ff49-nptbm evicted
    pod/firefly-iii-7bfd57ff49-k7rlx evicted
    pod/firefly-iii-7bfd57ff49-p9gsv evicted
    pod/firefly-iii-7bfd57ff49-pftg7 evicted
    pod/firefly-iii-7bfd57ff49-pnr9n evicted
    pod/firefly-iii-7bfd57ff49-pvrt5 evicted
    I0110 20:09:23.926166 1095415 request.go:690] Waited for 1.000164008s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.6:6443/api/v1/namespaces/firefly-iii/pods/firefly-iii-7bfd57ff49-qck44/eviction
    pod/firefly-iii-7bfd57ff49-qck44 evicted
    pod/atuin-cdc5879b8-8kl24 evicted
    pod/firefly-iii-7bfd57ff49-qfhzv evicted
    pod/firefly-iii-7bfd57ff49-qxgfc evicted
    pod/firefly-iii-7bfd57ff49-qzlfw evicted
    pod/firefly-iii-7bfd57ff49-r7v2f evicted
    pod/firefly-iii-7bfd57ff49-rltww evicted
    pod/firefly-iii-7bfd57ff49-rpfnc evicted
    pod/firefly-iii-7bfd57ff49-rx2kb evicted
    pod/firefly-iii-7bfd57ff49-rx92b evicted
    pod/firefly-iii-7bfd57ff49-s4ggm evicted
    pod/firefly-iii-7bfd57ff49-sbhrt evicted
    pod/firefly-iii-7bfd57ff49-sg6nw evicted
    pod/firefly-iii-7bfd57ff49-sj4vv evicted
    pod/firefly-iii-7bfd57ff49-sksd9 evicted
    pod/firefly-iii-7bfd57ff49-t2b58 evicted
    pod/firefly-iii-7bfd57ff49-t9fnn evicted
    pod/firefly-iii-7bfd57ff49-cb9bh evicted
    pod/firefly-iii-7bfd57ff49-tn62g evicted
    pod/firefly-iii-7bfd57ff49-tvcdg evicted
    pod/firefly-iii-7bfd57ff49-c9khh evicted
    pod/firefly-iii-7bfd57ff49-7m8b6 evicted
    pod/ingress-nginx-controller-6b58ffdc97-hd72r evicted
    pod/ingress-nginx-controller-6b58ffdc97-jdvng evicted
    pod/firefly-iii-7bfd57ff49-tzr6h evicted
    pod/ingress-nginx-controller-6b58ffdc97-jmg44 evicted
    pod/firefly-iii-7bfd57ff49-vvl94 evicted
    pod/firefly-iii-7bfd57ff49-vlqdn evicted
    pod/firefly-iii-7bfd57ff49-wv8qh evicted
    pod/ingress-nginx-controller-6b58ffdc97-jmsfn evicted
    pod/firefly-iii-7bfd57ff49-x6v5x evicted
    pod/firefly-iii-7bfd57ff49-xf5qv evicted
    pod/ingress-nginx-controller-6b58ffdc97-jtqtc evicted
    pod/ingress-nginx-controller-6b58ffdc97-k8xvt evicted
    pod/firefly-iii-7bfd57ff49-zcwd5 evicted
    pod/firefly-iii-7bfd57ff49-xh6s5 evicted
    pod/firefly-iii-7bfd57ff49-zp8fd evicted
    pod/ingress-nginx-controller-6b58ffdc97-khdf4 evicted
    pod/firefly-iii-7bfd57ff49-zr662 evicted
    pod/unifi-6678ccc684-7g9qb evicted
    pod/ingress-nginx-controller-6b58ffdc97-kprpk evicted
    pod/firefly-iii-7bfd57ff49-zrj82 evicted
    pod/ingress-nginx-controller-6b58ffdc97-lhffx evicted
    pod/firefly-iii-7bfd57ff49-zttt5 evicted
    pod/ingress-nginx-controller-6b58ffdc97-lxxqk evicted
    pod/ingress-nginx-controller-6b58ffdc97-mdvx4 evicted
    I0110 20:09:34.067126 1095415 request.go:690] Waited for 1.124324732s due to client-side throttling, not priority and fairness, request: GET:https://10.0.0.6:6443/api/v1/namespaces/firefly-iii/pods/firefly-iii-mysql-68f59d48f-k5mpw
    pod/ingress-nginx-controller-6b58ffdc97-mp7k9 evicted
    pod/homebox-7fb9d44d48-wxj6q evicted
    pod/ingress-nginx-controller-6b58ffdc97-mptfs evicted
    pod/ingress-nginx-admission-create-7zfzs evicted
    pod/ingress-nginx-controller-6b58ffdc97-nqnvc evicted
    pod/ingress-nginx-admission-patch-4tvvk evicted
    pod/ingress-nginx-controller-6b58ffdc97-pk8f4 evicted
    pod/ingress-nginx-controller-6b58ffdc97-244q5 evicted
    pod/ingress-nginx-controller-6b58ffdc97-qgb8v evicted
    pod/ingress-nginx-controller-6b58ffdc97-24z6b evicted
    pod/firefly-iii-mysql-68f59d48f-k5mpw evicted
    pod/ingress-nginx-controller-6b58ffdc97-r2t6n evicted
    pod/ingress-nginx-controller-6b58ffdc97-2brp6 evicted
    pod/ingress-nginx-controller-6b58ffdc97-r9kjw evicted
    pod/ingress-nginx-controller-6b58ffdc97-2g7t8 evicted
    pod/ingress-nginx-controller-6b58ffdc97-2g2c2 evicted
    pod/ingress-nginx-controller-6b58ffdc97-2k6pl evicted
    pod/ingress-nginx-controller-6b58ffdc97-rnz84 evicted
    pod/ingress-nginx-controller-6b58ffdc97-2mtgv evicted
    pod/ingress-nginx-controller-6b58ffdc97-rt5lm evicted
    pod/ingress-nginx-controller-6b58ffdc97-48gnx evicted
    pod/ingress-nginx-controller-6b58ffdc97-s5w6f evicted
    pod/ingress-nginx-controller-6b58ffdc97-4k4vc evicted
    pod/ingress-nginx-controller-6b58ffdc97-s989f evicted
    pod/ingress-nginx-controller-6b58ffdc97-snhd6 evicted
    pod/ingress-nginx-controller-6b58ffdc97-4rktx evicted
    pod/ingress-nginx-controller-6b58ffdc97-smqsc evicted
    pod/ingress-nginx-controller-6b58ffdc97-4sldn evicted
    pod/ingress-nginx-controller-6b58ffdc97-srhvn evicted
    pod/ingress-nginx-controller-6b58ffdc97-vdznd evicted
    pod/ingress-nginx-controller-6b58ffdc97-4wcq2 evicted
    pod/ingress-nginx-controller-6b58ffdc97-vftsh evicted
    pod/ingress-nginx-controller-6b58ffdc97-4wq8x evicted
    pod/ingress-nginx-controller-6b58ffdc97-vsxpk evicted
    pod/ingress-nginx-controller-6b58ffdc97-4ztpd evicted
    pod/ingress-nginx-controller-6b58ffdc97-vxbpz evicted
    pod/ingress-nginx-controller-6b58ffdc97-5m6w5 evicted
    pod/ingress-nginx-controller-6b58ffdc97-vz69m evicted
    pod/ingress-nginx-controller-6b58ffdc97-5rnzx evicted
    pod/ingress-nginx-controller-6b58ffdc97-wg556 evicted
    pod/ingress-nginx-controller-6b58ffdc97-66dpd evicted
    pod/ingress-nginx-controller-6b58ffdc97-x8sw4 evicted
    pod/ingress-nginx-controller-6b58ffdc97-6gdb7 evicted
    pod/ingress-nginx-controller-6b58ffdc97-xg4bx evicted
    pod/ingress-nginx-controller-6b58ffdc97-6tf9z evicted
    pod/ingress-nginx-controller-6b58ffdc97-xmfkv evicted
    pod/ingress-nginx-controller-6b58ffdc97-7h2qx evicted
    pod/ingress-nginx-controller-6b58ffdc97-xndqz evicted
    pod/ingress-nginx-controller-6b58ffdc97-87lgg evicted
    pod/ingress-nginx-controller-6b58ffdc97-826lj evicted
    I0110 20:09:44.125105 1095415 request.go:690] Waited for 21.198402997s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.6:6443/api/v1/namespaces/ingress-nginx/pods/ingress-nginx-controller-6b58ffdc97-zqd8d/eviction
    pod/ingress-nginx-controller-6b58ffdc97-8p9kb evicted
    pod/ingress-nginx-controller-6b58ffdc97-xtm2h evicted
    pod/ingress-nginx-controller-6b58ffdc97-8rs7g evicted
    pod/ingress-nginx-controller-6b58ffdc97-z5nr6 evicted
    pod/ingress-nginx-controller-6b58ffdc97-zp4tr evicted
    pod/ingress-nginx-controller-6b58ffdc97-9lljr evicted
    pod/ingress-nginx-controller-6b58ffdc97-zqd8d evicted
    pod/ingress-nginx-controller-6b58ffdc97-9lmrg evicted
    pod/ingress-nginx-controller-6b58ffdc97-zsfpv evicted
    pod/ingress-nginx-controller-6b58ffdc97-9mfzf evicted
    pod/ingress-nginx-controller-6b58ffdc97-9t8dd evicted
    pod/ingress-nginx-controller-6b58ffdc97-bqk9h evicted
    pod/ingress-nginx-controller-6b58ffdc97-bqw8m evicted
    pod/ingress-nginx-controller-6b58ffdc97-c77rw evicted
    pod/controller-68bf958bf9-79mpk evicted
    pod/ingress-nginx-controller-6b58ffdc97-c7dwq evicted
    pod/ingress-nginx-controller-6b58ffdc97-cmvt2 evicted
    pod/dashboard-metrics-scraper-7bc864c59-p27cg evicted
    pod/ingress-nginx-controller-6b58ffdc97-ddf2h evicted
    pod/ingress-nginx-controller-6b58ffdc97-dksm6 evicted
    pod/ingress-nginx-controller-6b58ffdc97-dkjmb evicted
    pod/ingress-nginx-controller-6b58ffdc97-f28xf evicted
    pod/nfd-node-feature-discovery-master-5f56c75d-xjkb8 evicted
    pod/ingress-nginx-controller-6b58ffdc97-f2k7w evicted
    pod/ingress-nginx-controller-6b58ffdc97-f5pft evicted
    I0110 20:09:54.125869 1095415 request.go:690] Waited for 31.198805794s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.6:6443/api/v1/namespaces/firefly-iii/pods/firefly-iii-7bfd57ff49-fzq24/eviction
    pod/mongo-d55fbb7f5-pvdfr evicted
    pod/ingress-nginx-controller-6b58ffdc97-ghjtz evicted
    pod/ingress-nginx-controller-6b58ffdc97-gs2w6 evicted
    pod/firefly-iii-7bfd57ff49-7qsnq evicted
    pod/firefly-iii-7bfd57ff49-ckx84 evicted
    pod/firefly-iii-7bfd57ff49-4wphp evicted
    pod/firefly-iii-7bfd57ff49-dgghp evicted
    pod/firefly-iii-7bfd57ff49-dzlgj evicted
    pod/firefly-iii-7bfd57ff49-5v9tk evicted
    pod/influxdb-57bddc4dc9-2z4v8 evicted
    pod/firefly-iii-7bfd57ff49-f75kw evicted
    pod/firefly-iii-7bfd57ff49-6jpm7 evicted
    pod/firefly-iii-7bfd57ff49-f8nm6 evicted
    pod/firefly-iii-7bfd57ff49-fphdc evicted
    pod/firefly-iii-7bfd57ff49-4lr52 evicted
    pod/firefly-iii-7bfd57ff49-fv6zx evicted
    pod/firefly-iii-7bfd57ff49-6jvv4 evicted
    pod/firefly-iii-7bfd57ff49-fvfwr evicted
    pod/coredns-787d4945fb-67z8g evicted
    pod/firefly-iii-7bfd57ff49-4n488 evicted
    pod/komga-549b6dd9b-4zvb5 evicted
    pod/firefly-iii-7bfd57ff49-fzq24 evicted
    pod/firefly-iii-7bfd57ff49-6ldtx evicted
    pod/metrics-server-74c749979-wd278 evicted
    pod/firefly-iii-7bfd57ff49-4nr7s evicted
    pod/firefly-iii-7bfd57ff49-g28xp evicted
    pod/plexserver-5dbdfc898b-9v2s8 evicted
    pod/firefly-iii-7bfd57ff49-jq85x evicted
    pod/firefly-iii-7bfd57ff49-h97wd evicted
    pod/firefly-iii-7bfd57ff49-m7gcs evicted
    pod/firefly-iii-7bfd57ff49-5st6w evicted
    pod/cert-manager-cainjector-56bbdd5c47-ltjgx evicted
    pod/firefly-iii-7bfd57ff49-jhhhc evicted
    pod/minecraft-server-bd48db4bd-wjc22 evicted
    pod/coredns-787d4945fb-gsx6h evicted
    pod/firefly-iii-7bfd57ff49-4gg7d evicted
    pod/kubernetes-dashboard-6c7ccbcf87-vwxnt evicted
    pod/grafana-7647f97d64-k8h5m evicted
    I0110 20:10:04.266451 1095415 request.go:690] Waited for 7.323749578s due to client-side throttling, not priority and fairness, request: GET:https://10.0.0.6:6443/api/v1/namespaces/firefly-iii/pods/firefly-iii-7bfd57ff49-22z7l
    pod/firefly-iii-7bfd57ff49-22z7l evicted
    pod/firefly-iii-7bfd57ff49-2xr55 evicted
    pod/inteldeviceplugins-controller-manager-7994555cdb-gjmfm evicted
    pod/firefly-iii-7bfd57ff49-2n2dz evicted
    pod/firefly-iii-7bfd57ff49-48vvc evicted
    pod/firefly-iii-5bd9bb7c4-vxv87 evicted
    pod/code-server-5768c9d997-cr858 evicted
    pod/firefly-iii-7bfd57ff49-928zt evicted
    pod/firefly-iii-7bfd57ff49-lrqfc evicted
    pod/firefly-iii-7bfd57ff49-8mrll evicted
    pod/firefly-iii-7bfd57ff49-hhv8z evicted
    pod/firefly-iii-7bfd57ff49-hlxqp evicted
    pod/navidrome-5f76b6ddf5-gcft4 evicted
    pod/firefly-iii-7bfd57ff49-m8q2t evicted
    pod/firefly-iii-7bfd57ff49-jf458 evicted
    pod/firefly-iii-7bfd57ff49-hpfx4 evicted
    pod/firefly-iii-7bfd57ff49-jgk98 evicted
    pod/firefly-iii-7bfd57ff49-bbpm2 evicted
    pod/firefly-iii-7bfd57ff49-bncmb evicted
    pod/firefly-iii-7bfd57ff49-c68ws evicted
    pod/firefly-iii-7bfd57ff49-76rlb evicted
    pod/cert-manager-64f9f45d6f-qx4hs evicted
    pod/cert-manager-webhook-d4f4545d7-tf4l6 evicted
    pod/audiobookshelf-987b955fb-64gxm evicted
    pod/ingress-nginx-controller-746f764ff4-x2xc2 evicted
    pod/kavita-766577cf45-dz9sl evicted
    pod/local-path-provisioner-8bc8875b-lbjh8 evicted
    ```

!!! note

    Local storage defined as `PersistentVolume` is **not** deleted:

    ``` console
    # du -sh /home/k8s/*
    210M    /home/k8s/atuin-server
    986M    /home/k8s/audiobookshelf
    4.7G    /home/k8s/code-server
    197M    /home/k8s/firefly-iii
    3.4M    /home/k8s/grafana
    236K    /home/k8s/homebox
    2.9G    /home/k8s/influxdb
    56M     /home/k8s/kavita
    14M     /home/k8s/komga
    9.0M    /home/k8s/local-path-storage
    1.6G    /home/k8s/minecraft-server
    37G     /home/k8s/minecraft-server-backups
    804M    /home/k8s/minecraft-server-spigot
    126M    /home/k8s/navidrome
    6.4G    /home/k8s/plexmediaserver
    404M    /home/k8s/unifi
    ```

At this point the Kubernetes cluster is running everything **but**
those pods that are not part of Kubernetes:

??? terminal "`# kubectl get deployments -A; kubectl get pods -A`"

    ``` console
    # kubectl get deployments -A
    NAMESPACE                NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
    atuin-server             atuin                                   0/1     1            0           494d
    audiobookshelf           audiobookshelf                          0/1     1            0           315d
    cert-manager             cert-manager                            0/1     1            0           635d
    cert-manager             cert-manager-cainjector                 0/1     1            0           635d
    cert-manager             cert-manager-webhook                    0/1     1            0           635d
    code-server              code-server                             0/1     1            0           631d
    default                  inteldeviceplugins-controller-manager   0/1     1            0           483d
    firefly-iii              firefly-iii                             0/1     1            0           236d
    firefly-iii              firefly-iii-mysql                       0/1     1            0           236d
    homebox                  homebox                                 0/1     1            0           185d
    ingress-nginx            ingress-nginx-controller                0/1     1            0           649d
    kavita                   kavita                                  0/1     1            0           235d
    komga                    komga                                   0/1     1            0           233d
    kube-system              coredns                                 0/2     2            0           659d
    kubernetes-dashboard     dashboard-metrics-scraper               0/1     1            0           649d
    kubernetes-dashboard     kubernetes-dashboard                    0/1     1            0           649d
    local-path-storage       local-path-provisioner                  0/1     1            0           635d
    metallb-system           controller                              0/1     1            0           649d
    metrics-server           metrics-server                          0/1     1            0           649d
    minecraft-server         minecraft-server                        0/1     1            0           14h
    monitoring               grafana                                 0/1     1            0           265d
    monitoring               influxdb                                0/1     1            0           265d
    navidrome                navidrome                               0/1     1            0           76d
    node-feature-discovery   nfd-node-feature-discovery-master       0/1     1            0           483d
    plexserver               plexserver                              0/1     1            0           467d
    unifi                    mongo                                   0/1     1            0           75d
    unifi                    unifi                                   0/1     1            0           9d

    # kubectl get pods -A
    NAMESPACE                NAME                                                     READY   STATUS    RESTARTS        AGE
    atuin-server             atuin-cdc5879b8-kcwh8                                    0/2     Pending   0               3m2s
    audiobookshelf           audiobookshelf-987b955fb-vsf4t                           0/1     Pending   0               2m29s
    cert-manager             cert-manager-64f9f45d6f-mqt2z                            0/1     Pending   0               2m29s
    cert-manager             cert-manager-cainjector-56bbdd5c47-hrmtb                 0/1     Pending   0               2m30s
    cert-manager             cert-manager-webhook-d4f4545d7-5mzt2                     0/1     Pending   0               2m29s
    code-server              code-server-5768c9d997-2ntzw                             0/1     Pending   0               2m17s
    default                  inteldeviceplugins-controller-manager-7994555cdb-k7bfz   0/2     Pending   0               2m28s
    firefly-iii              firefly-iii-5bd9bb7c4-ff4bs                              0/1     Pending   0               2m18s
    firefly-iii              firefly-iii-mysql-68f59d48f-dqhkh                        0/1     Pending   0               2m53s
    homebox                  homebox-7fb9d44d48-hg9jn                                 0/1     Pending   0               2m52s
    ingress-nginx            ingress-nginx-controller-746f764ff4-zf2fz                0/1     Pending   0               2m41s
    kavita                   kavita-766577cf45-5x2b2                                  0/1     Pending   0               2m40s
    komga                    komga-549b6dd9b-njkfc                                    0/1     Pending   0               2m40s
    kube-flannel             kube-flannel-ds-nrrg6                                    1/1     Running   131 (14h ago)   650d
    kube-system              coredns-787d4945fb-bqp8b                                 0/1     Pending   0               2m40s
    kube-system              coredns-787d4945fb-gmzww                                 0/1     Pending   0               2m40s
    kube-system              etcd-lexicon                                             1/1     Running   78 (14h ago)    293d
    kube-system              kube-apiserver-lexicon                                   1/1     Running   78 (14h ago)    293d
    kube-system              kube-controller-manager-lexicon                          1/1     Running   78 (14h ago)    293d
    kube-system              kube-proxy-qlt8c                                         1/1     Running   115 (14h ago)   659d
    kube-system              kube-scheduler-lexicon                                   1/1     Running   78 (14h ago)    293d
    kubernetes-dashboard     dashboard-metrics-scraper-7bc864c59-8zv47                0/1     Pending   0               2m39s
    kubernetes-dashboard     kubernetes-dashboard-6c7ccbcf87-78c58                    0/1     Pending   0               2m39s
    local-path-storage       local-path-provisioner-8bc8875b-n44d6                    0/1     Pending   0               2m39s
    metallb-system           controller-68bf958bf9-9f7f9                              0/1     Pending   0               2m38s
    metallb-system           speaker-t8f7k                                            1/1     Running   171 (14h ago)   649d
    metrics-server           metrics-server-74c749979-vggv9                           0/1     Pending   0               2m38s
    minecraft-server         minecraft-server-bd48db4bd-k4w4x                         0/1     Pending   0               2m37s
    monitoring               grafana-7647f97d64-ljdt9                                 0/1     Pending   0               2m37s
    monitoring               influxdb-57bddc4dc9-kmbw5                                0/1     Pending   0               2m37s
    monitoring               telegraf-ztg2t                                           1/1     Running   39 (14h ago)    87d
    navidrome                navidrome-5f76b6ddf5-4447v                               0/1     Pending   0               2m36s
    node-feature-discovery   nfd-node-feature-discovery-master-5f56c75d-88r54         0/1     Pending   0               2m36s
    node-feature-discovery   nfd-node-feature-discovery-worker-xmzwk                  0/1     Error     495 (79s ago)   483d
    plexserver               plexserver-5dbdfc898b-ng44d                              0/1     Pending   0               2m35s
    unifi                    mongo-d55fbb7f5-lssjb                                    0/1     Pending   0               2m35s
    unifi                    unifi-6678ccc684-6n9h7                                   0/1     Pending   0               2m58s
    ```

Now, to next step is to
[determine which version to upgrade to](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#determine-which-version-to-upgrade-to),
update the minor version in the repository configuration and
then find the latest patch version:

``` console
# vi /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /

# apt update
...
E: The repository 'https://pkgs.k8s.io/core:/stable:/v1.27/deb  InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

The first time around `apt update` failed because the relese key in
`/etc/apt/keyrings/kubernetes-apt-keyring.gpg` was obsolete; it had to be
replaced with the latest one 

``` console
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# apt update
```

``` console
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
[upgrade control plane nodes](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes) to:

??? terminal "`apt-get update && apt-get install -y kubeadm='1.27.16-*'`"

    ``` console
    # apt-mark unhold kubeadm && \
    apt-get update && apt-get install -y kubeadm='1.27.16-*' && \
    apt-mark hold kubeadm
    kubeadm was already not on hold.
    Get:1 https://cli.github.com/packages stable InRelease [3,917 B]
    Hit:2 http://ch.archive.ubuntu.com/ubuntu jammy InRelease                                      
    Hit:3 https://apt.releases.hashicorp.com jammy InRelease                                       
    Hit:4 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Hit:5 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Ign:6 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                     
    Hit:7 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:8 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease                             
    Hit:9 https://apt.grafana.com stable InRelease                                                 
    Err:1 https://cli.github.com/packages stable InRelease                                         
        The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:11 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease
    Hit:12 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease                         
    Hit:13 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease                      
    Hit:10 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  InRelease
    Hit:14 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease                       
    Hit:15 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release
    Err:10 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  InRelease
        The following signatures were invalid: EXPKEYSIG 234654DA9A296436 isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
    Hit:16 https://dl.ui.com/unifi/debian unifi-7.3 InRelease
    Err:17 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
        The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  InRelease: The following signatures were invalid: EXPKEYSIG 234654DA9A296436 isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://pkgs.k8s.io/core:/stable:/v1.27/deb/InRelease  The following signatures were invalid: EXPKEYSIG 234654DA9A296436 isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.27.16-1.1' (isv:kubernetes:core:stable:v1.27:pkgs.k8s.io [amd64]) for 'kubeadm'
    The following packages were automatically installed and are no longer required:
    linux-headers-5.15.0-122 linux-headers-5.15.0-124 linux-headers-5.15.0-125
    Use 'apt autoremove' to remove them.
    The following packages will be upgraded:
    kubeadm
    1 upgraded, 0 newly installed, 0 to remove and 16 not upgraded.
    Need to get 10.0 MB of archives.
    After this operation, 848 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  kubeadm 1.27.16-1.1 [10.0 MB]
    Fetched 10.0 MB in 1s (11.9 MB/s)
    (Reading database ... 280984 files and directories currently installed.)
    Preparing to unpack .../kubeadm_1.27.16-1.1_amd64.deb ...
    Unpacking kubeadm (1.27.16-1.1) over (1.26.15-1.1) ...
    Setting up kubeadm (1.27.16-1.1) ...
    Scanning processes...                                                                           
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    No services need to be restarted.

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubeadm set on hold.

    # kubeadm version
    kubeadm version: &version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.16", GitCommit:"cbb86e0d7f4a049666fac0551e8b02ef3d6c3d9a", GitTreeState:"clean", BuildDate:"2024-07-17T01:52:04Z", GoVersion:"go1.22.5", Compiler:"gc", Platform:"linux/amd64"}
    ```

Verify and apply the upgrade plan:

??? terminal "`# kubeadm upgrade plan && kubeadm upgrade apply v1.27.16`"

    ``` console
    # kubeadm upgrade plan
    [upgrade/config] Making sure the configuration is correct:
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [preflight] Running pre-flight checks.
    [upgrade] Running cluster health checks
    [upgrade] Fetching available versions to upgrade to
    [upgrade/versions] Cluster version: v1.26.3
    [upgrade/versions] kubeadm version: v1.27.16
    I0110 20:23:20.288112 1323315 version.go:256] remote version is much newer: v1.32.0; falling back to: stable-1.27
    [upgrade/versions] Target version: v1.27.16
    [upgrade/versions] Latest version in the v1.26 series: v1.26.15

    Upgrade to the latest version in the v1.26 series:

    COMPONENT                 CURRENT   TARGET
    kube-apiserver            v1.26.3   v1.26.15
    kube-controller-manager   v1.26.3   v1.26.15
    kube-scheduler            v1.26.3   v1.26.15
    kube-proxy                v1.26.3   v1.26.15
    CoreDNS                   v1.9.3    v1.10.1
    etcd                      3.5.6-0   3.5.12-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.26.15

    _____________________________________________________________________

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   CURRENT        TARGET
    kubelet     1 x v1.26.15   v1.27.16

    Upgrade to the latest stable version:

    COMPONENT                 CURRENT   TARGET
    kube-apiserver            v1.26.3   v1.27.16
    kube-controller-manager   v1.26.3   v1.27.16
    kube-scheduler            v1.26.3   v1.27.16
    kube-proxy                v1.26.3   v1.27.16
    CoreDNS                   v1.9.3    v1.10.1
    etcd                      3.5.6-0   3.5.12-0

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
    [upgrade/versions] Cluster version: v1.26.3
    [upgrade/versions] kubeadm version: v1.27.16
    [upgrade] Are you sure you want to proceed? [y/N]: y
    [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
    [upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
    [upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
    W0110 20:25:52.171444 1330873 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
    [upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.27.16" (timeout: 5m0s)...
    [upgrade/etcd] Upgrading to TLS for etcd
    [upgrade/staticpods] Preparing for "etcd" upgrade
    [upgrade/staticpods] Renewing etcd-server certificate
    [upgrade/staticpods] Renewing etcd-peer certificate
    [upgrade/staticpods] Renewing etcd-healthcheck-client certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-20-25-59/etcd.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=etcd
    [upgrade/staticpods] Component "etcd" upgraded successfully!
    [upgrade/etcd] Waiting for etcd to become available
    [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests2962201605"
    [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
    [upgrade/staticpods] Renewing apiserver certificate
    [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
    [upgrade/staticpods] Renewing front-proxy-client certificate
    [upgrade/staticpods] Renewing apiserver-etcd-client certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-20-25-59/kube-apiserver.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-apiserver
    [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
    [upgrade/staticpods] Renewing controller-manager.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-20-25-59/kube-controller-manager.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-controller-manager
    [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
    [upgrade/staticpods] Renewing scheduler.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-20-25-59/kube-scheduler.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-scheduler
    [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
    [upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config4258126724/config.yaml
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.27.16". Enjoy!

    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
    ```

Now that the control plane is updated, proceed to
[upgrade kubelet and kubectl](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl)

??? terminal "`apt-get update && apt-get install -y kubelet='1.27.16-*' kubectl='1.27.16-*'`"

    ``` console
    # apt-mark unhold kubelet kubectl && \
    apt-get update && apt-get install -y kubelet='1.27.16-*' kubectl='1.27.16-*' && \
    apt-mark hold kubelet kubectl
    kubelet was already not on hold.
    kubectl was already not on hold.
    Hit:1 http://ch.archive.ubuntu.com/ubuntu jammy InRelease
    Get:2 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]                     
    Hit:3 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Get:4 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Hit:5 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:6 https://apt.grafana.com stable InRelease                                                 
    Hit:7 https://apt.releases.hashicorp.com jammy InRelease                                       
    Get:8 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease [129 kB]                    
    Ign:10 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                    
    Hit:11 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease
    Err:4 https://cli.github.com/packages stable InRelease              
    The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:13 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease
    Hit:14 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease
    Hit:15 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease                       
    Hit:9 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  InRelease
    Hit:16 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release                      
    Hit:12 https://dl.ui.com/unifi/debian unifi-7.3 InRelease
    Err:9 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  InRelease
    The following signatures were invalid: EXPKEYSIG 234654DA9A296436 isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
    Err:17 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
    The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Fetched 261 kB in 1s (274 kB/s)
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  InRelease: The following signatures were invalid: EXPKEYSIG 234654DA9A296436 isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://pkgs.k8s.io/core:/stable:/v1.27/deb/InRelease  The following signatures were invalid: EXPKEYSIG 234654DA9A296436 isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.27.16-1.1' (isv:kubernetes:core:stable:v1.27:pkgs.k8s.io [amd64]) for 'kubelet'
    Selected version '1.27.16-1.1' (isv:kubernetes:core:stable:v1.27:pkgs.k8s.io [amd64]) for 'kubectl'
    The following packages were automatically installed and are no longer required:
        linux-headers-5.15.0-122 linux-headers-5.15.0-124 linux-headers-5.15.0-125
    Use 'apt autoremove' to remove them.
    The following packages will be upgraded:
        kubectl kubelet
    2 upgraded, 0 newly installed, 0 to remove and 14 not upgraded.
    Need to get 29.5 MB of archives.
    After this operation, 15.3 MB disk space will be freed.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  kubectl 1.27.16-1.1 [10.4 MB]
    Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.27/deb  kubelet 1.27.16-1.1 [19.2 MB]
    Fetched 29.5 MB in 1s (21.5 MB/s) 
    (Reading database ... 280984 files and directories currently installed.)
    Preparing to unpack .../kubectl_1.27.16-1.1_amd64.deb ...
    Unpacking kubectl (1.27.16-1.1) over (1.26.15-1.1) ...
    Preparing to unpack .../kubelet_1.27.16-1.1_amd64.deb ...
    Unpacking kubelet (1.27.16-1.1) over (1.26.15-1.1) ...
    dpkg: warning: unable to delete old directory '/etc/sysconfig': Directory not empty
    Setting up kubectl (1.27.16-1.1) ...
    Setting up kubelet (1.27.16-1.1) ...
    Scanning processes...                                                                           
    Scanning candidates...                                                                          
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    Restarting services...
    systemctl restart kubelet.service

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
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

??? terminal "`# kubectl uncordon lexicon`"

    ``` console
    # kubectl uncordon lexicon
    node/lexicon uncordoned
    # kubectl get pods -A
    NAMESPACE                NAME                                                     READY   STATUS              RESTARTS          AGE
    atuin-server             atuin-cdc5879b8-kcwh8                                    2/2     Running             0                 21m
    audiobookshelf           audiobookshelf-987b955fb-vsf4t                           0/1     ContainerCreating   0                 20m
    cert-manager             cert-manager-64f9f45d6f-mqt2z                            1/1     Running             0                 20m
    cert-manager             cert-manager-cainjector-56bbdd5c47-hrmtb                 1/1     Running             0                 20m
    cert-manager             cert-manager-webhook-d4f4545d7-5mzt2                     0/1     Running             0                 20m
    code-server              code-server-5768c9d997-2ntzw                             0/1     ContainerCreating   0                 20m
    default                  inteldeviceplugins-controller-manager-7994555cdb-k7bfz   2/2     Running             0                 20m
    firefly-iii              firefly-iii-5bd9bb7c4-ff4bs                              0/1     ContainerCreating   0                 20m
    firefly-iii              firefly-iii-mysql-68f59d48f-dqhkh                        1/1     Running             0                 20m
    homebox                  homebox-7fb9d44d48-hg9jn                                 1/1     Running             0                 20m
    ingress-nginx            ingress-nginx-controller-746f764ff4-zf2fz                0/1     Running             0                 20m
    kavita                   kavita-766577cf45-5x2b2                                  0/1     ContainerCreating   0                 20m
    komga                    komga-549b6dd9b-njkfc                                    0/1     ContainerCreating   0                 20m
    kube-flannel             kube-flannel-ds-nrrg6                                    1/1     Running             131 (14h ago)     650d
    kube-system              coredns-5d78c9869d-llpsv                                 1/1     Running             0                 102s
    kube-system              coredns-5d78c9869d-wn9xh                                 1/1     Running             0                 102s
    kube-system              coredns-787d4945fb-gmzww                                 1/1     Terminating         0                 20m
    kube-system              etcd-lexicon                                             1/1     Running             1 (47s ago)       43s
    kube-system              kube-apiserver-lexicon                                   1/1     Running             1 (46s ago)       43s
    kube-system              kube-controller-manager-lexicon                          1/1     Running             1 (47s ago)       47s
    kube-system              kube-proxy-pg4bx                                         1/1     Running             0                 102s
    kube-system              kube-scheduler-lexicon                                   1/1     Running             1 (47s ago)       47s
    kubernetes-dashboard     dashboard-metrics-scraper-7bc864c59-8zv47                1/1     Running             0                 20m
    kubernetes-dashboard     kubernetes-dashboard-6c7ccbcf87-78c58                    0/1     ContainerCreating   0                 20m
    local-path-storage       local-path-provisioner-8bc8875b-n44d6                    1/1     Running             0                 20m
    metallb-system           controller-68bf958bf9-9f7f9                              0/1     Running             0                 20m
    metallb-system           speaker-t8f7k                                            1/1     Running             171 (14h ago)     649d
    metrics-server           metrics-server-74c749979-vggv9                           0/1     Running             0                 20m
    minecraft-server         minecraft-server-bd48db4bd-k4w4x                         0/1     ContainerCreating   0                 20m
    monitoring               grafana-7647f97d64-ljdt9                                 1/1     Running             0                 20m
    monitoring               influxdb-57bddc4dc9-kmbw5                                1/1     Running             0                 20m
    monitoring               telegraf-ztg2t                                           1/1     Running             39 (14h ago)      87d
    navidrome                navidrome-5f76b6ddf5-4447v                               0/1     ContainerCreating   0                 20m
    node-feature-discovery   nfd-node-feature-discovery-master-5f56c75d-88r54         0/1     Running             0                 20m
    node-feature-discovery   nfd-node-feature-discovery-worker-xmzwk                  1/1     Running             501 (2m48s ago)   483d
    plexserver               plexserver-5dbdfc898b-ng44d                              0/1     ContainerCreating   0                 20m
    unifi                    mongo-d55fbb7f5-lssjb                                    1/1     Running             0                 20m
    unifi                    unifi-6678ccc684-6n9h7                                   1/1     Running             0                 21m
    ```

After a minute or two all the services are back up and running and the
Kubernetes dashboard shows all workloads green and OK :)

Now just repeat the same process again to upgrade to 1.28, then 1.29,
then 1.30, then 1.31, and finally 1.32. *Easy!*

### Upgrade to 1.28

[Upgrade version 1.27.x to version 1.28.x](https://v1-28.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) now, again starting by
determining which version to upgrade to and updating the minor version in the
repository configuration and then find the latest patch version:

``` console
# vi /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /

# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# apt update
# apt-cache madison kubeadm
   kubeadm | 1.28.15-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.14-2.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.13-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.12-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.11-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.10-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.9-2.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
```

Now, **before** updating `kubeadm`, drain the code again:

??? terminal "`# kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon`"

    ``` console
    # kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon
    node/lexicon cordoned
    Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-nrrg6, kube-system/kube-proxy-pg4bx, metallb-system/speaker-t8f7k, monitoring/telegraf-ztg2t, node-feature-discovery/nfd-node-feature-discovery-worker-xmzwk
    evicting pod atuin-server/atuin-cdc5879b8-kcwh8
    evicting pod cert-manager/cert-manager-64f9f45d6f-mqt2z
    evicting pod audiobookshelf/audiobookshelf-987b955fb-vsf4t
    evicting pod komga/komga-549b6dd9b-njkfc
    evicting pod cert-manager/cert-manager-cainjector-56bbdd5c47-hrmtb
    evicting pod cert-manager/cert-manager-webhook-d4f4545d7-5mzt2
    evicting pod kube-system/coredns-5d78c9869d-llpsv
    evicting pod code-server/code-server-5768c9d997-2ntzw
    evicting pod unifi/unifi-6678ccc684-6n9h7
    evicting pod default/inteldeviceplugins-controller-manager-7994555cdb-k7bfz
    evicting pod kube-system/coredns-5d78c9869d-wn9xh
    evicting pod firefly-iii/firefly-iii-5bd9bb7c4-ff4bs
    evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-8zv47
    evicting pod firefly-iii/firefly-iii-mysql-68f59d48f-dqhkh
    evicting pod local-path-storage/local-path-provisioner-8bc8875b-n44d6
    evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-78c58
    evicting pod metrics-server/metrics-server-74c749979-vggv9
    evicting pod metallb-system/controller-68bf958bf9-9f7f9
    evicting pod monitoring/grafana-7647f97d64-ljdt9
    evicting pod minecraft-server/minecraft-server-bd48db4bd-k4w4x
    evicting pod node-feature-discovery/nfd-node-feature-discovery-master-5f56c75d-88r54
    evicting pod homebox/homebox-7fb9d44d48-hg9jn
    evicting pod navidrome/navidrome-5f76b6ddf5-4447v
    evicting pod ingress-nginx/ingress-nginx-controller-746f764ff4-zf2fz
    evicting pod plexserver/plexserver-5dbdfc898b-ng44d
    evicting pod unifi/mongo-d55fbb7f5-lssjb
    evicting pod monitoring/influxdb-57bddc4dc9-kmbw5
    evicting pod kavita/kavita-766577cf45-5x2b2
    I0110 20:54:21.549407 1964930 request.go:696] Waited for 1.000157756s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.6:6443/api/v1/namespaces/cert-manager/pods/cert-manager-webhook-d4f4545d7-5mzt2/eviction
    pod/navidrome-5f76b6ddf5-4447v evicted
    pod/homebox-7fb9d44d48-hg9jn evicted
    pod/cert-manager-64f9f45d6f-mqt2z evicted
    pod/atuin-cdc5879b8-kcwh8 evicted
    pod/metrics-server-74c749979-vggv9 evicted
    pod/cert-manager-cainjector-56bbdd5c47-hrmtb evicted
    pod/inteldeviceplugins-controller-manager-7994555cdb-k7bfz evicted
    pod/influxdb-57bddc4dc9-kmbw5 evicted
    pod/firefly-iii-5bd9bb7c4-ff4bs evicted
    pod/dashboard-metrics-scraper-7bc864c59-8zv47 evicted
    pod/komga-549b6dd9b-njkfc evicted
    pod/plexserver-5dbdfc898b-ng44d evicted
    pod/code-server-5768c9d997-2ntzw evicted
    pod/firefly-iii-mysql-68f59d48f-dqhkh evicted
    pod/cert-manager-webhook-d4f4545d7-5mzt2 evicted
    pod/grafana-7647f97d64-ljdt9 evicted
    pod/minecraft-server-bd48db4bd-k4w4x evicted
    pod/kubernetes-dashboard-6c7ccbcf87-78c58 evicted
    pod/audiobookshelf-987b955fb-vsf4t evicted
    pod/mongo-d55fbb7f5-lssjb evicted
    pod/controller-68bf958bf9-9f7f9 evicted
    pod/nfd-node-feature-discovery-master-5f56c75d-88r54 evicted
    pod/coredns-5d78c9869d-llpsv evicted
    pod/coredns-5d78c9869d-wn9xh evicted
    pod/unifi-6678ccc684-6n9h7 evicted
    pod/ingress-nginx-controller-746f764ff4-zf2fz evicted
    pod/local-path-provisioner-8bc8875b-n44d6 evicted
    pod/kavita-766577cf45-5x2b2 evicted
    node/lexicon drained
    ```

The latest patch version is 1.28.**10** and that is the one to
[upgrade control plane nodes](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes) to:

??? terminal "`apt-get update && apt-get install -y kubeadm='1.28.15-*'`"

    ``` console
    # apt-mark unhold kubeadm && \
    apt-get update && apt-get install -y kubeadm='1.28.15-*' && \
    apt-mark hold kubeadm
    Canceled hold on kubeadm.
    Hit:1 http://ch.archive.ubuntu.com/ubuntu jammy InRelease
    Hit:2 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Hit:3 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:4 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Get:5 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Ign:6 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                     
    Hit:7 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease                             
    Hit:8 https://apt.releases.hashicorp.com jammy InRelease                                       
    Hit:9 https://apt.grafana.com stable InRelease                                                 
    Hit:12 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease                        
    Hit:13 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease                 
    Err:5 https://cli.github.com/packages stable InRelease                                 
    The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:14 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease                      
    Hit:15 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release                      
    Hit:16 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease                       
    Hit:10 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.28/deb  InRelease
    Hit:11 https://dl.ui.com/unifi/debian unifi-7.3 InRelease         
    Err:17 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
    The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Fetched 3,917 B in 1s (5,193 B/s)
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.28.15-1.1' (isv:kubernetes:core:stable:v1.28:pkgs.k8s.io [amd64]) for 'kubeadm'
    The following packages were automatically installed and are no longer required:
    linux-headers-5.15.0-122 linux-headers-5.15.0-124 linux-headers-5.15.0-125
    Use 'apt autoremove' to remove them.
    The following packages will be upgraded:
    kubeadm
    1 upgraded, 0 newly installed, 0 to remove and 15 not upgraded.
    Need to get 10.1 MB of archives.
    After this operation, 369 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.28/deb  kubeadm 1.28.15-1.1 [10.1 MB]
    Fetched 10.1 MB in 1s (15.7 MB/s)
    (Reading database ... 280984 files and directories currently installed.)
    Preparing to unpack .../kubeadm_1.28.15-1.1_amd64.deb ...
    Unpacking kubeadm (1.28.15-1.1) over (1.28.10-1.1) ...
    Setting up kubeadm (1.28.15-1.1) ...
    Scanning processes...                                                                           
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    No services need to be restarted.

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubeadm set on hold.
    ```

Verify and apply the upgrade plan:

??? terminal "`# kubeadm upgrade plan && kubeadm upgrade apply v1.28.15`"

    ``` console
    # kubeadm upgrade plan
    [upgrade/config] Making sure the configuration is correct:
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [preflight] Running pre-flight checks.
    [upgrade] Running cluster health checks
    [upgrade] Fetching available versions to upgrade to
    [upgrade/versions] Cluster version: v1.27.16
    [upgrade/versions] kubeadm version: v1.28.15
    I0110 21:01:58.619294 2087758 version.go:256] remote version is much newer: v1.32.0; falling back to: stable-1.28
    [upgrade/versions] Target version: v1.28.15
    [upgrade/versions] Latest version in the v1.27 series: v1.27.16

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   CURRENT        TARGET
    kubelet     1 x v1.27.16   v1.28.15

    Upgrade to the latest stable version:

    COMPONENT                 CURRENT    TARGET
    kube-apiserver            v1.27.16   v1.28.15
    kube-controller-manager   v1.27.16   v1.28.15
    kube-scheduler            v1.27.16   v1.28.15
    kube-proxy                v1.27.16   v1.28.15
    CoreDNS                   v1.10.1    v1.10.1
    etcd                      3.5.12-0   3.5.15-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.28.15

    _____________________________________________________________________


    The table below shows the current state of component configs as understood by this version of kubeadm.
    Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
    resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
    upgrade to is denoted in the "PREFERRED VERSION" column.

    API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
    kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
    kubelet.config.k8s.io     v1beta1           v1beta1             no
    _____________________________________________________________________


    # kubeadm upgrade apply v1.28.15
    [upgrade/config] Making sure the configuration is correct:
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [preflight] Running pre-flight checks.
    [upgrade] Running cluster health checks
    [upgrade/version] You have chosen to change the cluster version to "v1.28.15"
    [upgrade/versions] Cluster version: v1.27.16
    [upgrade/versions] kubeadm version: v1.28.15
    [upgrade] Are you sure you want to proceed? [y/N]: y
    [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
    [upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
    [upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
    W0110 21:03:05.773743 2099300 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
    [upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.28.15" (timeout: 5m0s)...
    [upgrade/etcd] Upgrading to TLS for etcd
    [upgrade/staticpods] Preparing for "etcd" upgrade
    [upgrade/staticpods] Renewing etcd-server certificate
    [upgrade/staticpods] Renewing etcd-peer certificate
    [upgrade/staticpods] Renewing etcd-healthcheck-client certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-03-08/etcd.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=etcd
    [upgrade/staticpods] Component "etcd" upgraded successfully!
    [upgrade/etcd] Waiting for etcd to become available
    [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests3220413090"
    [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
    [upgrade/staticpods] Renewing apiserver certificate
    [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
    [upgrade/staticpods] Renewing front-proxy-client certificate
    [upgrade/staticpods] Renewing apiserver-etcd-client certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-03-08/kube-apiserver.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-apiserver
    [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
    [upgrade/staticpods] Renewing controller-manager.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-03-08/kube-controller-manager.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-controller-manager
    [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
    [upgrade/staticpods] Renewing scheduler.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-03-08/kube-scheduler.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-scheduler
    [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
    [upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config4047502448/config.yaml
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.28.15". Enjoy!

    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
    ```

Now that the control plane is updated, proceed to
[upgrade kubelet and kubectl](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl)

??? terminal "`apt-get update && apt-get install -y kubelet='1.28.15-*' kubectl='1.28.15-*'`"

    ``` console
    # apt-mark unhold kubelet kubectl && \
    apt-get update && apt-get install -y kubelet='1.28.15-*' kubectl='1.28.15-*' && \
    apt-mark hold kubelet kubectl
    Canceled hold on kubelet.
    Canceled hold on kubectl.
    Hit:1 https://download.docker.com/linux/ubuntu jammy InRelease
    Hit:2 http://ch.archive.ubuntu.com/ubuntu jammy InRelease                                      
    Ign:3 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                     
    Get:4 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Hit:5 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Hit:6 https://apt.releases.hashicorp.com jammy InRelease                                       
    Hit:7 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:8 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease                             
    Hit:9 https://apt.grafana.com stable InRelease                                                 
    Err:4 https://cli.github.com/packages stable InRelease                                         
    The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:12 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release                  
    Hit:10 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.28/deb  InRelease
    Hit:11 https://dl.ui.com/unifi/debian unifi-7.3 InRelease                                      
    Err:13 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
    The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Hit:14 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease
    Hit:15 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease
    Hit:16 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease
    Hit:17 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease
    Fetched 3,917 B in 1s (3,152 B/s)
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.28.15-1.1' (isv:kubernetes:core:stable:v1.28:pkgs.k8s.io [amd64]) for 'kubelet'
    Selected version '1.28.15-1.1' (isv:kubernetes:core:stable:v1.28:pkgs.k8s.io [amd64]) for 'kubectl'
    The following packages were automatically installed and are no longer required:
    ebtables linux-headers-5.15.0-122 linux-headers-5.15.0-124 linux-headers-5.15.0-125 socat
    Use 'apt autoremove' to remove them.
    The following packages will be upgraded:
    kubectl kubelet
    2 upgraded, 0 newly installed, 0 to remove and 13 not upgraded.
    Need to get 30.0 MB of archives.
    After this operation, 2,494 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.28/deb  kubectl 1.28.15-1.1 [10.4 MB]
    Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.28/deb  kubelet 1.28.15-1.1 [19.6 MB]
    Fetched 30.0 MB in 1s (29.2 MB/s)
    (Reading database ... 280984 files and directories currently installed.)
    Preparing to unpack .../kubectl_1.28.15-1.1_amd64.deb ...
    Unpacking kubectl (1.28.15-1.1) over (1.27.16-1.1) ...
    Preparing to unpack .../kubelet_1.28.15-1.1_amd64.deb ...
    Unpacking kubelet (1.28.15-1.1) over (1.27.16-1.1) ...
    Setting up kubectl (1.28.15-1.1) ...
    Setting up kubelet (1.28.15-1.1) ...
    Scanning processes...                                                                           
    Scanning candidates...                                                                          
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    Restarting services...
    systemctl restart kubelet.service

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubelet set on hold.
    kubectl set on hold.

    # systemctl daemon-reload
    # systemctl restart kubelet

    # kubectl  version --output=yaml
    clientVersion:
        buildDate: "2024-10-22T20:34:56Z"
        compiler: gc
        gitCommit: 841856557ef0f6a399096c42635d114d6f2cf7f4
        gitTreeState: clean
        gitVersion: v1.28.15
        goVersion: go1.22.8
        major: "1"
        minor: "28"
        platform: linux/amd64
    kustomizeVersion: v5.0.4-0.20230601165947-6ce0bf390ce3
    serverVersion:
        buildDate: "2024-10-22T20:26:27Z"
        compiler: gc
        gitCommit: 841856557ef0f6a399096c42635d114d6f2cf7f4
        gitTreeState: clean
        gitVersion: v1.28.15
        goVersion: go1.22.8
        major: "1"
        minor: "28"
        platform: linux/amd64
    ```

Finally, bring the node back online by marking it schedulable:

??? terminal "`# kubectl uncordon lexicon`"

    ``` console
    # kubectl uncordon lexicon
    node/lexicon uncordoned
    # kubectl get pods -A
    NAMESPACE                NAME                                                     READY   STATUS    RESTARTS          AGE
    atuin-server             atuin-cdc5879b8-4ssbz                                    0/2     Pending   0                 15m
    audiobookshelf           audiobookshelf-987b955fb-2vkq7                           0/1     Pending   0                 15m
    cert-manager             cert-manager-64f9f45d6f-hjkd6                            0/1     Pending   0                 15m
    cert-manager             cert-manager-cainjector-56bbdd5c47-2nxfs                 0/1     Pending   0                 15m
    cert-manager             cert-manager-webhook-d4f4545d7-jjzpj                     0/1     Pending   0                 15m
    code-server              code-server-5768c9d997-5mmz6                             0/1     Pending   0                 15m
    default                  inteldeviceplugins-controller-manager-7994555cdb-jvxtk   0/2     Pending   0                 14m
    firefly-iii              firefly-iii-5bd9bb7c4-vh9qx                              0/1     Pending   0                 14m
    firefly-iii              firefly-iii-mysql-68f59d48f-2zlb4                        0/1     Pending   0                 15m
    homebox                  homebox-7fb9d44d48-ml6hp                                 0/1     Pending   0                 15m
    ingress-nginx            ingress-nginx-controller-746f764ff4-hfl47                0/1     Pending   0                 15m
    kavita                   kavita-766577cf45-2hdrc                                  0/1     Pending   0                 15m
    komga                    komga-549b6dd9b-fxbfr                                    0/1     Pending   0                 15m
    kube-flannel             kube-flannel-ds-nrrg6                                    1/1     Running   131 (15h ago)     650d
    kube-system              coredns-5d78c9869d-vmf8t                                 0/1     Pending   0                 15m
    kube-system              coredns-5d78c9869d-x5tpm                                 0/1     Pending   0                 15m
    kube-system              etcd-lexicon                                             1/1     Running   1 (20s ago)       16s
    kube-system              kube-apiserver-lexicon                                   1/1     Running   1 (20s ago)       16s
    kube-system              kube-controller-manager-lexicon                          1/1     Running   1 (20s ago)       16s
    kube-system              kube-proxy-7rfxg                                         1/1     Running   0                 3m36s
    kube-system              kube-scheduler-lexicon                                   1/1     Running   1 (20s ago)       16s
    kubernetes-dashboard     dashboard-metrics-scraper-7bc864c59-mpm5d                0/1     Pending   0                 15m
    kubernetes-dashboard     kubernetes-dashboard-6c7ccbcf87-z2798                    0/1     Pending   0                 15m
    local-path-storage       local-path-provisioner-8bc8875b-lwc8t                    0/1     Pending   0                 15m
    metallb-system           controller-68bf958bf9-dblz8                              0/1     Pending   0                 15m
    metallb-system           speaker-t8f7k                                            1/1     Running   171 (15h ago)     649d
    metrics-server           metrics-server-74c749979-84jk5                           0/1     Pending   0                 15m
    minecraft-server         minecraft-server-bd48db4bd-hscp6                         0/1     Pending   0                 15m
    monitoring               grafana-7647f97d64-k57tg                                 0/1     Pending   0                 15m
    monitoring               influxdb-57bddc4dc9-562fw                                0/1     Pending   0                 15m
    monitoring               telegraf-ztg2t                                           1/1     Running   39 (15h ago)      87d
    navidrome                navidrome-5f76b6ddf5-92k2z                               0/1     Pending   0                 15m
    node-feature-discovery   nfd-node-feature-discovery-master-5f56c75d-rkgrm         0/1     Pending   0                 15m
    node-feature-discovery   nfd-node-feature-discovery-worker-xmzwk                  1/1     Running   509 (2m59s ago)   483d
    plexserver               plexserver-5dbdfc898b-nsmjw                              0/1     Pending   0                 15m
    unifi                    mongo-d55fbb7f5-td9zr                                    0/1     Pending   0                 15m
    unifi                    unifi-6678ccc684-2zsbg                                   0/1     Pending   0                 15m
    ```

After a minute or two all the services are back up and running and the
Kubernetes dashboard shows all workloads green and OK again.

### Upgrade to 1.29

[Upgrade version 1.28.x to version 1.29.x](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) now, again starting by
determining which version to upgrade to and updating the minor version in the
repository configuration and then find the latest patch version:

``` console
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# vi /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /

# apt update
# apt-cache madison kubeadm
   kubeadm | 1.29.12-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.11-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.10-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.4-2.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.29.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
```

Now, **before** updating `kubeadm`, drain the code again:

??? terminal "`# kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon`"

    ``` console
    # kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon
    node/lexicon cordoned
    Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-nrrg6, kube-system/kube-proxy-7rfxg, metallb-system/speaker-t8f7k, monitoring/telegraf-ztg2t, node-feature-discovery/nfd-node-feature-discovery-worker-xmzwk
    evicting pod audiobookshelf/audiobookshelf-987b955fb-2vkq7
    evicting pod cert-manager/cert-manager-64f9f45d6f-hjkd6
    evicting pod navidrome/navidrome-5f76b6ddf5-92k2z
    evicting pod komga/komga-549b6dd9b-fxbfr
    evicting pod atuin-server/atuin-cdc5879b8-4ssbz
    evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-z2798
    evicting pod cert-manager/cert-manager-cainjector-56bbdd5c47-2nxfs
    evicting pod node-feature-discovery/nfd-node-feature-discovery-master-5f56c75d-rkgrm
    evicting pod local-path-storage/local-path-provisioner-8bc8875b-lwc8t
    evicting pod cert-manager/cert-manager-webhook-d4f4545d7-jjzpj
    evicting pod metallb-system/controller-68bf958bf9-dblz8
    evicting pod code-server/code-server-5768c9d997-5mmz6
    evicting pod kube-system/coredns-5d78c9869d-vmf8t
    evicting pod default/inteldeviceplugins-controller-manager-7994555cdb-jvxtk
    evicting pod minecraft-server/minecraft-server-bd48db4bd-hscp6
    evicting pod firefly-iii/firefly-iii-5bd9bb7c4-vh9qx
    evicting pod monitoring/grafana-7647f97d64-k57tg
    evicting pod firefly-iii/firefly-iii-mysql-68f59d48f-2zlb4
    evicting pod homebox/homebox-7fb9d44d48-ml6hp
    evicting pod monitoring/influxdb-57bddc4dc9-562fw
    evicting pod unifi/unifi-6678ccc684-2zsbg
    evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-mpm5d
    evicting pod metrics-server/metrics-server-74c749979-84jk5
    evicting pod plexserver/plexserver-5dbdfc898b-nsmjw
    evicting pod kube-system/coredns-5d78c9869d-x5tpm
    evicting pod ingress-nginx/ingress-nginx-controller-746f764ff4-hfl47
    evicting pod unifi/mongo-d55fbb7f5-td9zr
    evicting pod kavita/kavita-766577cf45-2hdrc
    pod/controller-68bf958bf9-dblz8 evicted
    I0110 21:17:25.198339 2388267 request.go:697] Waited for 1.200180309s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.6:6443/api/v1/namespaces/cert-manager/pods/cert-manager-cainjector-56bbdd5c47-2nxfs/eviction
    pod/cert-manager-64f9f45d6f-hjkd6 evicted
    pod/homebox-7fb9d44d48-ml6hp evicted
    pod/influxdb-57bddc4dc9-562fw evicted
    pod/cert-manager-webhook-d4f4545d7-jjzpj evicted
    pod/code-server-5768c9d997-5mmz6 evicted
    pod/cert-manager-cainjector-56bbdd5c47-2nxfs evicted
    pod/nfd-node-feature-discovery-master-5f56c75d-rkgrm evicted
    pod/firefly-iii-5bd9bb7c4-vh9qx evicted
    pod/navidrome-5f76b6ddf5-92k2z evicted
    pod/kubernetes-dashboard-6c7ccbcf87-z2798 evicted
    pod/atuin-cdc5879b8-4ssbz evicted
    pod/grafana-7647f97d64-k57tg evicted
    pod/inteldeviceplugins-controller-manager-7994555cdb-jvxtk evicted
    pod/audiobookshelf-987b955fb-2vkq7 evicted
    pod/unifi-6678ccc684-2zsbg evicted
    pod/minecraft-server-bd48db4bd-hscp6 evicted
    pod/firefly-iii-mysql-68f59d48f-2zlb4 evicted
    pod/komga-549b6dd9b-fxbfr evicted
    pod/dashboard-metrics-scraper-7bc864c59-mpm5d evicted
    pod/metrics-server-74c749979-84jk5 evicted
    pod/coredns-5d78c9869d-vmf8t evicted
    pod/mongo-d55fbb7f5-td9zr evicted
    pod/plexserver-5dbdfc898b-nsmjw evicted
    pod/coredns-5d78c9869d-x5tpm evicted
    pod/ingress-nginx-controller-746f764ff4-hfl47 evicted
    pod/local-path-provisioner-8bc8875b-lwc8t evicted
    pod/kavita-766577cf45-2hdrc evicted
    node/lexicon drained
    ```

The latest patch version is 1.29.**12** and that is the one to
[upgrade control plane nodes](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes) to:

??? terminal "`apt-get update && apt-get install -y kubeadm='1.29.12-*'`"

    ``` console
    # apt-mark unhold kubeadm && \
    apt-get update && apt-get install -y kubeadm='1.29.12-*' && \
    apt-mark hold kubeadm
    Canceled hold on kubeadm.
    Hit:1 http://ch.archive.ubuntu.com/ubuntu jammy InRelease
    Ign:2 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                     
    Hit:3 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Hit:4 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:5 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease                             
    Get:6 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Hit:7 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Hit:8 https://apt.releases.hashicorp.com jammy InRelease                                       
    Hit:10 https://apt.grafana.com stable InRelease                                                
    Hit:11 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease                   
    Hit:13 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release                      
    Hit:14 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease                         
    Hit:9 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.29/deb  InRelease
    Hit:15 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease
    Hit:16 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease
    Err:6 https://cli.github.com/packages stable InRelease
        The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:12 https://dl.ui.com/unifi/debian unifi-7.3 InRelease
    Err:17 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
        The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.29.12-1.1' (isv:kubernetes:core:stable:v1.29:pkgs.k8s.io [amd64]) for 'kubeadm'
    The following packages were automatically installed and are no longer required:
        ebtables linux-headers-5.15.0-122 linux-headers-5.15.0-124 linux-headers-5.15.0-125 socat
    Use 'apt autoremove' to remove them.
    The following packages will be upgraded:
        kubeadm
    1 upgraded, 0 newly installed, 0 to remove and 17 not upgraded.
    Need to get 10.2 MB of archives.
    After this operation, 81.9 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.29/deb  kubeadm 1.29.12-1.1 [10.2 MB]
    Fetched 10.2 MB in 1s (12.2 MB/s)
    (Reading database ... 280984 files and directories currently installed.)
    Preparing to unpack .../kubeadm_1.29.12-1.1_amd64.deb ...
    Unpacking kubeadm (1.29.12-1.1) over (1.28.15-1.1) ...
    Setting up kubeadm (1.29.12-1.1) ...
    Scanning processes...                                                                           
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    No services need to be restarted.

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubeadm set on hold.
    ```

Verify and apply the upgrade plan:

??? terminal "`# kubeadm upgrade plan && kubeadm upgrade apply v1.29.12`"

    ``` console
    # kubeadm upgrade plan
    [upgrade/config] Making sure the configuration is correct:
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [preflight] Running pre-flight checks.
    [upgrade] Running cluster health checks
    [upgrade] Fetching available versions to upgrade to
    [upgrade/versions] Cluster version: v1.28.15
    [upgrade/versions] kubeadm version: v1.29.12
    I0110 21:19:52.303946 2429342 version.go:256] remote version is much newer: v1.32.0; falling back to: stable-1.29
    [upgrade/versions] Target version: v1.29.12
    [upgrade/versions] Latest version in the v1.28 series: v1.28.15

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   CURRENT        TARGET
    kubelet     1 x v1.28.15   v1.29.12

    Upgrade to the latest stable version:

    COMPONENT                 CURRENT    TARGET
    kube-apiserver            v1.28.15   v1.29.12
    kube-controller-manager   v1.28.15   v1.29.12
    kube-scheduler            v1.28.15   v1.29.12
    kube-proxy                v1.28.15   v1.29.12
    CoreDNS                   v1.10.1    v1.11.1
    etcd                      3.5.15-0   3.5.16-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.29.12

    _____________________________________________________________________


    The table below shows the current state of component configs as understood by this version of kubeadm.
    Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
    resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
    upgrade to is denoted in the "PREFERRED VERSION" column.

    API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
    kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
    kubelet.config.k8s.io     v1beta1           v1beta1             no
    _____________________________________________________________________

    # kubeadm upgrade apply v1.29.12
    [upgrade/config] Making sure the configuration is correct:
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [preflight] Running pre-flight checks.
    [upgrade] Running cluster health checks
    [upgrade/version] You have chosen to change the cluster version to "v1.29.12"
    [upgrade/versions] Cluster version: v1.28.15
    [upgrade/versions] kubeadm version: v1.29.12
    [upgrade] Are you sure you want to proceed? [y/N]: y
    [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
    [upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
    [upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
    W0110 21:21:18.273461 2445192 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
    [upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.29.12" (timeout: 5m0s)...
    [upgrade/etcd] Upgrading to TLS for etcd
    [upgrade/staticpods] Preparing for "etcd" upgrade
    [upgrade/staticpods] Renewing etcd-server certificate
    [upgrade/staticpods] Renewing etcd-peer certificate
    [upgrade/staticpods] Renewing etcd-healthcheck-client certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-21-21/etcd.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=etcd
    [upgrade/staticpods] Component "etcd" upgraded successfully!
    [upgrade/etcd] Waiting for etcd to become available
    [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests797080025"
    [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
    [upgrade/staticpods] Renewing apiserver certificate
    [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
    [upgrade/staticpods] Renewing front-proxy-client certificate
    [upgrade/staticpods] Renewing apiserver-etcd-client certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-21-21/kube-apiserver.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-apiserver
    [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
    [upgrade/staticpods] Renewing controller-manager.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-21-21/kube-controller-manager.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-controller-manager
    [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
    [upgrade/staticpods] Renewing scheduler.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-21-21/kube-scheduler.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-scheduler
    [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
    [upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config2507373492/config.yaml
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubeconfig] Writing "admin.conf" kubeconfig file
    [kubeconfig] Writing "super-admin.conf" kubeconfig file
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.29.12". Enjoy!

    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
    ```

Now that the control plane is updated, proceed to
[upgrade kubelet and kubectl](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl)

??? terminal "`apt-get update && apt-get install -y kubelet='1.29.12-*' kubectl='1.29.12-*'`"

    ``` console
    # apt-mark unhold kubelet kubectl && \
    apt-get update && apt-get install -y kubelet='1.29.12-*' kubectl='1.29.12-*' && \
    apt-mark hold kubelet kubectl
    Canceled hold on kubelet.
    Canceled hold on kubectl.
    Hit:1 http://ch.archive.ubuntu.com/ubuntu jammy InRelease
    Hit:2 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Hit:3 https://apt.releases.hashicorp.com jammy InRelease                                       
    Get:4 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Hit:5 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:6 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Ign:7 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                     
    Hit:8 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease                             
    Hit:9 https://apt.grafana.com stable InRelease                                                 
    Hit:10 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease                        
    Hit:12 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease                    
    Hit:14 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease              
    Err:4 https://cli.github.com/packages stable InRelease                                 
        The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:15 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease               
    Hit:16 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release                      
    Hit:11 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.29/deb  InRelease
    Hit:13 https://dl.ui.com/unifi/debian unifi-7.3 InRelease                           
    Err:17 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
        The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.29.12-1.1' (isv:kubernetes:core:stable:v1.29:pkgs.k8s.io [amd64]) for 'kubelet'
    Selected version '1.29.12-1.1' (isv:kubernetes:core:stable:v1.29:pkgs.k8s.io [amd64]) for 'kubectl'
    The following packages were automatically installed and are no longer required:
        ebtables linux-headers-5.15.0-122 linux-headers-5.15.0-124 linux-headers-5.15.0-125 socat
    Use 'apt autoremove' to remove them.
    The following packages will be upgraded:
        kubectl kubelet
    2 upgraded, 0 newly installed, 0 to remove and 15 not upgraded.
    Need to get 30.5 MB of archives.
    After this operation, 2,556 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.29/deb  kubectl 1.29.12-1.1 [10.6 MB]
    Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.29/deb  kubelet 1.29.12-1.1 [19.9 MB]
    Fetched 30.5 MB in 1s (27.2 MB/s) 
    (Reading database ... 280984 files and directories currently installed.)
    Preparing to unpack .../kubectl_1.29.12-1.1_amd64.deb ...
    Unpacking kubectl (1.29.12-1.1) over (1.28.15-1.1) ...
    Preparing to unpack .../kubelet_1.29.12-1.1_amd64.deb ...
    Unpacking kubelet (1.29.12-1.1) over (1.28.15-1.1) ...
    Setting up kubectl (1.29.12-1.1) ...
    Setting up kubelet (1.29.12-1.1) ...
    Scanning processes...                                                                           
    Scanning candidates...                                                                          
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    Restarting services...
    systemctl restart kubelet.service

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubelet set on hold.
    kubectl set on hold.

    # systemctl daemon-reload
    # systemctl restart kubelet

    # kubectl  version --output=yaml
    clientVersion:
        buildDate: "2024-12-10T11:36:13Z"
        compiler: gc
        gitCommit: 9253c9bda3d8bd76848bb4a21b309c28c0aab2f7
        gitTreeState: clean
        gitVersion: v1.29.12
        goVersion: go1.22.9
        major: "1"
        minor: "29"
        platform: linux/amd64
    kustomizeVersion: v5.0.4-0.20230601165947-6ce0bf390ce3
    serverVersion:
        buildDate: "2024-12-10T11:27:08Z"
        compiler: gc
        gitCommit: 9253c9bda3d8bd76848bb4a21b309c28c0aab2f7
        gitTreeState: clean
        gitVersion: v1.29.12
        goVersion: go1.22.9
        major: "1"
        minor: "29"
        platform: linux/amd64
    ```

Finally, bring the node back online by marking it schedulable:

??? terminal "`# kubectl uncordon lexicon`"

    ``` console
    # kubectl uncordon lexicon
    node/lexicon uncordoned
    # kubectl get pods -A
    NAMESPACE                NAME                                                     READY   STATUS              RESTARTS        AGE
    atuin-server             atuin-cdc5879b8-vf6pn                                    2/2     Running             0               10m
    audiobookshelf           audiobookshelf-987b955fb-c9vrr                           0/1     ContainerCreating   0               10m
    cert-manager             cert-manager-64f9f45d6f-mxjxc                            0/1     ContainerCreating   0               10m
    cert-manager             cert-manager-cainjector-56bbdd5c47-bznk2                 1/1     Running             0               10m
    cert-manager             cert-manager-webhook-d4f4545d7-zbq67                     0/1     ContainerCreating   0               10m
    code-server              code-server-5768c9d997-rhzmx                             0/1     ContainerCreating   0               10m
    default                  inteldeviceplugins-controller-manager-7994555cdb-g58dm   0/2     ContainerCreating   0               10m
    firefly-iii              firefly-iii-5bd9bb7c4-wqrqb                              0/1     ContainerCreating   0               10m
    firefly-iii              firefly-iii-mysql-68f59d48f-8n6ff                        0/1     ContainerCreating   0               10m
    homebox                  homebox-7fb9d44d48-mb62g                                 0/1     ContainerCreating   0               10m
    ingress-nginx            ingress-nginx-controller-746f764ff4-kjfjv                0/1     Running             0               10m
    kavita                   kavita-766577cf45-285x5                                  0/1     ContainerCreating   0               10m
    komga                    komga-549b6dd9b-bfz6n                                    0/1     ContainerCreating   0               10m
    kube-flannel             kube-flannel-ds-nrrg6                                    1/1     Running             131 (15h ago)   650d
    kube-system              coredns-5d78c9869d-p96rz                                 0/1     Terminating         0               10m
    kube-system              coredns-76f75df574-4qtvd                                 1/1     Running             0               3m34s
    kube-system              coredns-76f75df574-9p9xg                                 0/1     ContainerCreating   0               3m34s
    kube-system              etcd-lexicon                                             1/1     Running             0               5m7s
    kube-system              kube-apiserver-lexicon                                   1/1     Running             0               4m25s
    kube-system              kube-controller-manager-lexicon                          1/1     Running             0               4m4s
    kube-system              kube-proxy-xv4gm                                         1/1     Running             0               3m34s
    kube-system              kube-scheduler-lexicon                                   1/1     Running             0               3m48s
    kubernetes-dashboard     dashboard-metrics-scraper-7bc864c59-5hnt8                1/1     Running             0               10m
    kubernetes-dashboard     kubernetes-dashboard-6c7ccbcf87-q6z7q                    0/1     ContainerCreating   0               10m
    local-path-storage       local-path-provisioner-8bc8875b-wlbt9                    0/1     ContainerCreating   0               10m
    metallb-system           controller-68bf958bf9-2tztx                              0/1     ContainerCreating   0               10m
    metallb-system           speaker-t8f7k                                            1/1     Running             171 (15h ago)   649d
    metrics-server           metrics-server-74c749979-ngtp7                           0/1     ContainerCreating   0               10m
    minecraft-server         minecraft-server-bd48db4bd-nbzwd                         0/1     ContainerCreating   0               10m
    monitoring               grafana-7647f97d64-6vgdh                                 0/1     ContainerCreating   0               10m
    monitoring               influxdb-57bddc4dc9-gw2lx                                0/1     ContainerCreating   0               10m
    monitoring               telegraf-ztg2t                                           1/1     Running             39 (15h ago)    87d
    navidrome                navidrome-5f76b6ddf5-z9gmk                               0/1     ContainerCreating   0               10m
    node-feature-discovery   nfd-node-feature-discovery-master-5f56c75d-fv894         0/1     Running             0               10m
    node-feature-discovery   nfd-node-feature-discovery-worker-xmzwk                  1/1     Running             517 (29s ago)   483d
    plexserver               plexserver-5dbdfc898b-fmdb7                              0/1     ContainerCreating   0               10m
    unifi                    mongo-d55fbb7f5-478g7                                    0/1     ContainerCreating   0               10m
    unifi                    unifi-6678ccc684-lv6lz                                   0/1     ContainerCreating   0               10m
    ```

After a minute or two all the services are back up and running and the
Kubernetes dashboard shows all workloads green and OK again.

### Upgrade to 1.30

[Upgrade version 1.29.x to version 1.30.x](https://v1-30.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) now, again starting by
determining which version to upgrade to and updating the minor version in the
repository configuration and then find the latest patch version:

``` console
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# vi /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /

# apt update
# apt-cache madison kubeadm
   kubeadm | 1.30.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
```

Now, **before** updating `kubeadm`, drain the code again:

??? terminal "`# kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon`"

    ``` console
    # kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon
    node/lexicon cordoned
    Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-nrrg6, kube-system/kube-proxy-xv4gm, metallb-system/speaker-t8f7k, monitoring/telegraf-ztg2t, node-feature-discovery/nfd-node-feature-discovery-worker-xmzwk
    evicting pod audiobookshelf/audiobookshelf-987b955fb-c9vrr
    evicting pod atuin-server/atuin-cdc5879b8-vf6pn
    evicting pod unifi/unifi-6678ccc684-lv6lz
    evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-5hnt8
    evicting pod kube-system/coredns-76f75df574-9p9xg
    evicting pod monitoring/grafana-7647f97d64-6vgdh
    evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-q6z7q
    evicting pod local-path-storage/local-path-provisioner-8bc8875b-wlbt9
    evicting pod monitoring/influxdb-57bddc4dc9-gw2lx
    evicting pod metallb-system/controller-68bf958bf9-2tztx
    evicting pod cert-manager/cert-manager-64f9f45d6f-mxjxc
    evicting pod navidrome/navidrome-5f76b6ddf5-z9gmk
    evicting pod metrics-server/metrics-server-74c749979-ngtp7
    evicting pod cert-manager/cert-manager-cainjector-56bbdd5c47-bznk2
    evicting pod plexserver/plexserver-5dbdfc898b-fmdb7
    evicting pod minecraft-server/minecraft-server-bd48db4bd-nbzwd
    evicting pod unifi/mongo-d55fbb7f5-478g7
    evicting pod firefly-iii/firefly-iii-mysql-68f59d48f-8n6ff
    evicting pod kavita/kavita-766577cf45-285x5
    evicting pod komga/komga-549b6dd9b-bfz6n
    evicting pod cert-manager/cert-manager-webhook-d4f4545d7-zbq67
    evicting pod kube-system/coredns-76f75df574-4qtvd
    evicting pod node-feature-discovery/nfd-node-feature-discovery-master-5f56c75d-fv894
    evicting pod code-server/code-server-5768c9d997-rhzmx
    evicting pod homebox/homebox-7fb9d44d48-mb62g
    evicting pod firefly-iii/firefly-iii-5bd9bb7c4-wqrqb
    evicting pod default/inteldeviceplugins-controller-manager-7994555cdb-g58dm
    evicting pod ingress-nginx/ingress-nginx-controller-746f764ff4-kjfjv
    I0110 21:33:20.594704 2688967 request.go:697] Waited for 1.200299817s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.6:6443/api/v1/namespaces/node-feature-discovery/pods/nfd-node-feature-discovery-master-5f56c75d-fv894/eviction
    pod/audiobookshelf-987b955fb-c9vrr evicted
    pod/grafana-7647f97d64-6vgdh evicted
    pod/kubernetes-dashboard-6c7ccbcf87-q6z7q evicted
    pod/nfd-node-feature-discovery-master-5f56c75d-fv894 evicted
    pod/atuin-cdc5879b8-vf6pn evicted
    pod/controller-68bf958bf9-2tztx evicted
    pod/komga-549b6dd9b-bfz6n evicted
    pod/cert-manager-64f9f45d6f-mxjxc evicted
    pod/firefly-iii-5bd9bb7c4-wqrqb evicted
    pod/dashboard-metrics-scraper-7bc864c59-5hnt8 evicted
    pod/code-server-5768c9d997-rhzmx evicted
    pod/navidrome-5f76b6ddf5-z9gmk evicted
    pod/influxdb-57bddc4dc9-gw2lx evicted
    pod/cert-manager-webhook-d4f4545d7-zbq67 evicted
    pod/metrics-server-74c749979-ngtp7 evicted
    pod/inteldeviceplugins-controller-manager-7994555cdb-g58dm evicted
    pod/cert-manager-cainjector-56bbdd5c47-bznk2 evicted
    pod/firefly-iii-mysql-68f59d48f-8n6ff evicted
    pod/minecraft-server-bd48db4bd-nbzwd evicted
    pod/unifi-6678ccc684-lv6lz evicted
    pod/plexserver-5dbdfc898b-fmdb7 evicted
    pod/mongo-d55fbb7f5-478g7 evicted
    pod/homebox-7fb9d44d48-mb62g evicted
    pod/coredns-76f75df574-9p9xg evicted
    pod/coredns-76f75df574-4qtvd evicted
    pod/ingress-nginx-controller-746f764ff4-kjfjv evicted
    pod/local-path-provisioner-8bc8875b-wlbt9 evicted
    pod/kavita-766577cf45-285x5 evicted
    node/lexicon drained
    ```

The latest patch version is 1.30.**8** and that is the one to
[upgrade control plane nodes](https://v1-30.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes) to:

??? terminal "`apt-get update && apt-get install -y kubeadm='1.30.8-*'`"

    ``` console
    # apt-mark unhold kubeadm && \
    apt-get update && apt-get install -y kubeadm='1.30.8-*' && \
    apt-mark hold kubeadm
    Canceled hold on kubeadm.
    Hit:1 http://ch.archive.ubuntu.com/ubuntu jammy InRelease
    Hit:2 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Get:3 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Hit:4 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:5 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Hit:6 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease                             
    Hit:7 https://apt.releases.hashicorp.com jammy InRelease                                       
    Ign:8 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                     
    Hit:9 https://apt.grafana.com stable InRelease                                                 
    Err:3 https://cli.github.com/packages stable InRelease                                         
        The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:12 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release                      
    Hit:10 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease
    Hit:11 https://dl.ui.com/unifi/debian unifi-7.3 InRelease                              
    Err:13 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
        The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Hit:14 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease
    Hit:15 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease
    Hit:16 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease
    Hit:17 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.30.8-1.1' (isv:kubernetes:core:stable:v1.30:pkgs.k8s.io [amd64]) for 'kubeadm'
    The following additional packages will be installed:
        cri-tools
    The following packages will be upgraded:
        cri-tools kubeadm
    2 upgraded, 0 newly installed, 0 to remove and 16 not upgraded.
    Need to get 31.7 MB of archives.
    After this operation, 4,985 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  cri-tools 1.30.1-1.1 [21.3 MB]
    Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  kubeadm 1.30.8-1.1 [10.4 MB]
    Fetched 31.7 MB in 1s (30.8 MB/s)
    (Reading database ... 224773 files and directories currently installed.)
    Preparing to unpack .../cri-tools_1.30.1-1.1_amd64.deb ...
    Unpacking cri-tools (1.30.1-1.1) over (1.28.0-1.1) ...
    Preparing to unpack .../kubeadm_1.30.8-1.1_amd64.deb ...
    Unpacking kubeadm (1.30.8-1.1) over (1.29.12-1.1) ...
    Setting up cri-tools (1.30.1-1.1) ...
    Setting up kubeadm (1.30.8-1.1) ...
    Scanning processes...                                                                           
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    No services need to be restarted.

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubeadm set on hold.
    ```

Verify and apply the upgrade plan:

??? terminal "`# kubeadm upgrade plan && kubeadm upgrade apply v1.30.8`"

    ``` console
    # kubeadm upgrade plan
    [preflight] Running pre-flight checks.
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [upgrade] Running cluster health checks
    W0110 21:35:27.210495 2728007 health.go:132] The preflight check "CreateJob" was skipped because there are no schedulable Nodes in the cluster.
    [upgrade] Fetching available versions to upgrade to
    [upgrade/versions] Cluster version: 1.29.12
    [upgrade/versions] kubeadm version: v1.30.8
    I0110 21:35:27.800090 2728007 version.go:256] remote version is much newer: v1.32.0; falling back to: stable-1.30
    [upgrade/versions] Target version: v1.30.8
    [upgrade/versions] Latest version in the v1.29 series: v1.29.12

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   NODE      CURRENT    TARGET
    kubelet     lexicon   v1.29.12   v1.30.8

    Upgrade to the latest stable version:

    COMPONENT                 NODE      CURRENT    TARGET
    kube-apiserver            lexicon   v1.29.12   v1.30.8
    kube-controller-manager   lexicon   v1.29.12   v1.30.8
    kube-scheduler            lexicon   v1.29.12   v1.30.8
    kube-proxy                          1.29.12    v1.30.8
    CoreDNS                             v1.11.1    v1.11.3
    etcd                      lexicon   3.5.16-0   3.5.15-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.30.8

    _____________________________________________________________________


    The table below shows the current state of component configs as understood by this version of kubeadm.
    Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
    resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
    upgrade to is denoted in the "PREFERRED VERSION" column.

    API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
    kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
    kubelet.config.k8s.io     v1beta1           v1beta1             no
    _____________________________________________________________________


    # kubeadm upgrade apply v1.30.8
    [preflight] Running pre-flight checks.
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [upgrade] Running cluster health checks
    W0110 21:36:16.532765 2741120 health.go:132] The preflight check "CreateJob" was skipped because there are no schedulable Nodes in the cluster.
    [upgrade/version] You have chosen to change the cluster version to "v1.30.8"
    [upgrade/versions] Cluster version: v1.29.12
    [upgrade/versions] kubeadm version: v1.30.8
    [upgrade] Are you sure you want to proceed? [y/N]: y
    [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
    [upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
    [upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
    W0110 21:36:19.260976 2741120 checks.go:844] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "registry.k8s.io/pause:3.9" as the CRI sandbox image.
    [upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.30.8" (timeout: 5m0s)...
    [upgrade/etcd] Upgrading to TLS for etcd
    [upgrade/etcd] Non fatal issue encountered during upgrade: the desired etcd version "3.5.15-0" is older than the currently installed "3.5.16-0". Skipping etcd upgrade
    [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests2222122892"
    [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
    [upgrade/staticpods] Renewing apiserver certificate
    [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
    [upgrade/staticpods] Renewing front-proxy-client certificate
    [upgrade/staticpods] Renewing apiserver-etcd-client certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-36-33/kube-apiserver.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-apiserver
    [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
    [upgrade/staticpods] Renewing controller-manager.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-36-33/kube-controller-manager.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-controller-manager
    [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
    [upgrade/staticpods] Renewing scheduler.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-21-36-33/kube-scheduler.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-scheduler
    [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
    [upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config995930464/config.yaml
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.30.8". Enjoy!

    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
    ```

Now that the control plane is updated, proceed to
[upgrade kubelet and kubectl](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl)

??? terminal "`apt-get update && apt-get install -y kubelet='1.30.8-*' kubectl='1.30.8-*'`"

    ``` console
    # apt-mark unhold kubelet kubectl && \
    apt-get update && apt-get install -y kubelet='1.30.8-*' kubectl='1.30.8-*' && \
    apt-mark hold kubelet kubectl
    Canceled hold on kubelet.
    Canceled hold on kubectl.
    Hit:1 http://ch.archive.ubuntu.com/ubuntu jammy InRelease
    Get:2 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Hit:3 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Hit:4 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:5 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Ign:6 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                     
    Hit:7 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease                             
    Hit:8 https://apt.releases.hashicorp.com jammy InRelease                                       
    Hit:9 https://apt.grafana.com stable InRelease                                                 
    Err:2 https://cli.github.com/packages stable InRelease                                         
        The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:12 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease
    Hit:13 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release
    Hit:14 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease                         
    Hit:15 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease                      
    Hit:10 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease
    Hit:11 https://dl.ui.com/unifi/debian unifi-7.3 InRelease           
    Hit:16 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease
    Err:17 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
        The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.30.8-1.1' (isv:kubernetes:core:stable:v1.30:pkgs.k8s.io [amd64]) for 'kubelet'
    Selected version '1.30.8-1.1' (isv:kubernetes:core:stable:v1.30:pkgs.k8s.io [amd64]) for 'kubectl'
    The following packages will be upgraded:
        kubectl kubelet
    2 upgraded, 0 newly installed, 0 to remove and 14 not upgraded.
    Need to get 28.9 MB of archives.
    After this operation, 11.2 MB disk space will be freed.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  kubectl 1.30.8-1.1 [10.8 MB]
    Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  kubelet 1.30.8-1.1 [18.1 MB]
    Fetched 28.9 MB in 1s (29.5 MB/s)
    (Reading database ... 224773 files and directories currently installed.)
    Preparing to unpack .../kubectl_1.30.8-1.1_amd64.deb ...
    Unpacking kubectl (1.30.8-1.1) over (1.29.12-1.1) ...
    Preparing to unpack .../kubelet_1.30.8-1.1_amd64.deb ...
    Unpacking kubelet (1.30.8-1.1) over (1.29.12-1.1) ...
    Setting up kubectl (1.30.8-1.1) ...
    Setting up kubelet (1.30.8-1.1) ...
    Scanning processes...                                                                           
    Scanning candidates...                                                                          
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    Restarting services...
    systemctl restart kubelet.service

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubelet set on hold.
    kubectl set on hold.

    # systemctl daemon-reload
    # systemctl restart kubelet

    # kubectl  version --output=yaml
    clientVersion:
        buildDate: "2024-12-10T11:31:16Z"
        compiler: gc
        gitCommit: 354eac776046f4268e9989b21f8d1bba06033379
        gitTreeState: clean
        gitVersion: v1.30.8
        goVersion: go1.22.9
        major: "1"
        minor: "30"
        platform: linux/amd64
    kustomizeVersion: v5.0.4-0.20230601165947-6ce0bf390ce3
    serverVersion:
        buildDate: "2024-12-10T11:25:06Z"
        compiler: gc
        gitCommit: 354eac776046f4268e9989b21f8d1bba06033379
        gitTreeState: clean
        gitVersion: v1.30.8
        goVersion: go1.22.9
        major: "1"
        minor: "30"
        platform: linux/amd64
    ```

Finally, bring the node back online by marking it schedulable:

??? terminal "`# kubectl uncordon lexicon`"

    ``` console
    # kubectl uncordon lexicon
    node/lexicon uncordoned
    # kubectl get pods -A
    NAMESPACE                NAME                                                     READY   STATUS              RESTARTS        AGE
    atuin-server             atuin-cdc5879b8-9zg5b                                    2/2     Running             0               7m48s
    audiobookshelf           audiobookshelf-987b955fb-2m7m2                           0/1     ContainerCreating   0               7m48s
    cert-manager             cert-manager-64f9f45d6f-vdwvx                            1/1     Running             0               7m46s
    cert-manager             cert-manager-cainjector-56bbdd5c47-mlr6n                 1/1     Running             0               7m45s
    cert-manager             cert-manager-webhook-d4f4545d7-2gt2b                     0/1     Running             0               7m45s
    code-server              code-server-5768c9d997-6xs5t                             1/1     Running             0               7m47s
    default                  inteldeviceplugins-controller-manager-7994555cdb-8bf65   2/2     Running             0               7m35s
    firefly-iii              firefly-iii-5bd9bb7c4-9f5c8                              0/1     ContainerCreating   0               7m46s
    firefly-iii              firefly-iii-mysql-68f59d48f-5t9gs                        0/1     ContainerCreating   0               7m48s
    homebox                  homebox-7fb9d44d48-xwznt                                 0/1     ContainerCreating   0               7m44s
    ingress-nginx            ingress-nginx-controller-746f764ff4-mkkzh                0/1     Running             0               7m48s
    kavita                   kavita-766577cf45-l2jf6                                  0/1     ContainerCreating   0               7m47s
    komga                    komga-549b6dd9b-z5bwv                                    0/1     ContainerCreating   0               7m48s
    kube-flannel             kube-flannel-ds-nrrg6                                    1/1     Running             131 (15h ago)   650d
    kube-system              coredns-55cb58b774-cmrlk                                 1/1     Running             0               3m1s
    kube-system              coredns-55cb58b774-rckht                                 1/1     Running             0               3m1s
    kube-system              coredns-76f75df574-vldnl                                 1/1     Terminating         0               7m47s
    kube-system              etcd-lexicon                                             1/1     Running             1 (105s ago)    104s
    kube-system              kube-apiserver-lexicon                                   1/1     Running             1 (104s ago)    104s
    kube-system              kube-controller-manager-lexicon                          1/1     Running             1 (104s ago)    100s
    kube-system              kube-proxy-4gpjg                                         1/1     Running             0               3m1s
    kube-system              kube-scheduler-lexicon                                   1/1     Running             1 (104s ago)    104s
    kubernetes-dashboard     dashboard-metrics-scraper-7bc864c59-vwznk                1/1     Running             0               7m48s
    kubernetes-dashboard     kubernetes-dashboard-6c7ccbcf87-j46fg                    0/1     ContainerCreating   0               7m48s
    local-path-storage       local-path-provisioner-8bc8875b-7pn9w                    1/1     Running             0               7m48s
    metallb-system           controller-68bf958bf9-lhckf                              0/1     Running             0               7m46s
    metallb-system           speaker-t8f7k                                            1/1     Running             171 (15h ago)   649d
    metrics-server           metrics-server-74c749979-wx77h                           0/1     Running             0               7m35s
    minecraft-server         minecraft-server-bd48db4bd-jkdz4                         0/1     ContainerCreating   0               7m48s
    monitoring               grafana-7647f97d64-rnz5j                                 1/1     Running             0               7m48s
    monitoring               influxdb-57bddc4dc9-q2z99                                1/1     Running             0               7m47s
    monitoring               telegraf-ztg2t                                           1/1     Running             39 (15h ago)    87d
    navidrome                navidrome-5f76b6ddf5-lc47q                               0/1     ContainerCreating   0               7m45s
    node-feature-discovery   nfd-node-feature-discovery-master-5f56c75d-dh6fc         0/1     Running             0               7m47s
    node-feature-discovery   nfd-node-feature-discovery-worker-xmzwk                  1/1     Running             522 (44s ago)   483d
    plexserver               plexserver-5dbdfc898b-2lfsq                              0/1     ContainerCreating   0               7m45s
    unifi                    mongo-d55fbb7f5-wz592                                    1/1     Running             0               7m44s
    unifi                    unifi-6678ccc684-6vvgc                                   1/1     Running             0               7m48s
    ```

After a minute or two all the services are back up and running and the
Kubernetes dashboard shows all workloads green and OK again.

### Upgrade to 1.31

[Upgrade version 1.30.x to version 1.31.x](https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) now, again starting by
determining which version to upgrade to and updating the minor version in the
repository configuration and then find the latest patch version:

``` console
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# vi /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /

# apt update
# apt-cache madison kubeadm
   kubeadm | 1.31.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.31/deb  Packages
   kubeadm | 1.31.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.31/deb  Packages
   kubeadm | 1.31.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.31/deb  Packages
   kubeadm | 1.31.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.31/deb  Packages
   kubeadm | 1.31.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.31/deb  Packages
```

Now, **before** updating `kubeadm`, drain the code again:

??? terminal "`# kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon`"

    ``` console
    # kubectl drain --ignore-daemonsets --delete-emptydir-data lexicon
    node/lexicon cordoned
    Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-nrrg6, kube-system/kube-proxy-4gpjg, metallb-system/speaker-t8f7k, monitoring/telegraf-ztg2t, node-feature-discovery/nfd-node-feature-discovery-worker-xmzwk
    evicting pod unifi/unifi-6678ccc684-6vvgc
    evicting pod atuin-server/atuin-cdc5879b8-9zg5b
    evicting pod audiobookshelf/audiobookshelf-987b955fb-2m7m2
    evicting pod cert-manager/cert-manager-64f9f45d6f-vdwvx
    evicting pod cert-manager/cert-manager-cainjector-56bbdd5c47-mlr6n
    evicting pod cert-manager/cert-manager-webhook-d4f4545d7-2gt2b
    evicting pod kubernetes-dashboard/kubernetes-dashboard-6c7ccbcf87-j46fg
    evicting pod kubernetes-dashboard/dashboard-metrics-scraper-7bc864c59-vwznk
    evicting pod monitoring/influxdb-57bddc4dc9-q2z99
    evicting pod code-server/code-server-5768c9d997-6xs5t
    evicting pod navidrome/navidrome-5f76b6ddf5-lc47q
    evicting pod homebox/homebox-7fb9d44d48-xwznt
    evicting pod ingress-nginx/ingress-nginx-controller-746f764ff4-mkkzh
    evicting pod local-path-storage/local-path-provisioner-8bc8875b-7pn9w
    evicting pod kavita/kavita-766577cf45-l2jf6
    evicting pod default/inteldeviceplugins-controller-manager-7994555cdb-8bf65
    evicting pod node-feature-discovery/nfd-node-feature-discovery-master-5f56c75d-dh6fc
    evicting pod komga/komga-549b6dd9b-z5bwv
    evicting pod firefly-iii/firefly-iii-5bd9bb7c4-9f5c8
    evicting pod metrics-server/metrics-server-74c749979-wx77h
    evicting pod plexserver/plexserver-5dbdfc898b-2lfsq
    evicting pod kube-system/coredns-55cb58b774-rckht
    evicting pod metallb-system/controller-68bf958bf9-lhckf
    evicting pod kube-system/coredns-55cb58b774-cmrlk
    evicting pod unifi/mongo-d55fbb7f5-wz592
    evicting pod firefly-iii/firefly-iii-mysql-68f59d48f-5t9gs
    evicting pod monitoring/grafana-7647f97d64-rnz5j
    I0110 22:43:43.853741 4143116 request.go:697] Waited for 1.000055459s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.6:6443/api/v1/namespaces/default/pods/inteldeviceplugins-controller-manager-7994555cdb-8bf65/eviction
    pod/navidrome-5f76b6ddf5-lc47q evicted
    pod/audiobookshelf-987b955fb-2m7m2 evicted
    pod/cert-manager-cainjector-56bbdd5c47-mlr6n evicted
    pod/atuin-cdc5879b8-9zg5b evicted
    pod/code-server-5768c9d997-6xs5t evicted
    pod/cert-manager-64f9f45d6f-vdwvx evicted
    pod/cert-manager-webhook-d4f4545d7-2gt2b evicted
    pod/dashboard-metrics-scraper-7bc864c59-vwznk evicted
    pod/nfd-node-feature-discovery-master-5f56c75d-dh6fc evicted
    pod/influxdb-57bddc4dc9-q2z99 evicted
    pod/homebox-7fb9d44d48-xwznt evicted
    pod/firefly-iii-5bd9bb7c4-9f5c8 evicted
    pod/metrics-server-74c749979-wx77h evicted
    pod/kubernetes-dashboard-6c7ccbcf87-j46fg evicted
    pod/inteldeviceplugins-controller-manager-7994555cdb-8bf65 evicted
    pod/controller-68bf958bf9-lhckf evicted
    pod/firefly-iii-mysql-68f59d48f-5t9gs evicted
    pod/mongo-d55fbb7f5-wz592 evicted
    pod/grafana-7647f97d64-rnz5j evicted
    pod/unifi-6678ccc684-6vvgc evicted
    pod/komga-549b6dd9b-z5bwv evicted
    pod/plexserver-5dbdfc898b-2lfsq evicted
    pod/coredns-55cb58b774-cmrlk evicted
    pod/coredns-55cb58b774-rckht evicted
    pod/ingress-nginx-controller-746f764ff4-mkkzh evicted
    pod/kavita-766577cf45-l2jf6 evicted
    pod/local-path-provisioner-8bc8875b-7pn9w evicted
    node/lexicon drained
    ```

The latest patch version is 1.31.**4** and that is the one to
[upgrade control plane nodes](https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes) to:

??? terminal "`apt-get update && apt-get install -y kubeadm='1.31.4-*'`"

    ``` console
    # apt-mark unhold kubeadm && \
    apt-get update && apt-get install -y kubeadm='1.31.4-*' && \
    apt-mark hold kubeadm
    Canceled hold on kubeadm.
    Hit:1 http://ch.archive.ubuntu.com/ubuntu jammy InRelease
    Hit:2 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Hit:3 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Hit:4 https://apt.releases.hashicorp.com jammy InRelease                                       
    Ign:5 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease                     
    Hit:6 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                            
    Hit:7 https://apt.grafana.com stable InRelease                                                 
    Hit:8 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease                             
    Get:9 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Get:12 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease [7,565 B]              
    Get:13 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease [7,456 B]
    Hit:14 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release             
    Get:15 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease [7,450 B]            
    Get:16 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease [7,449 B]             
    Hit:10 https://dl.ui.com/unifi/debian unifi-7.3 InRelease                                      
    Hit:11 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.31/deb  InRelease
    Err:9 https://cli.github.com/packages stable InRelease
        The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Err:17 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
        The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Fetched 29.9 kB in 1s (29.8 kB/s)
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.31.4-1.1' (isv:kubernetes:core:stable:v1.31:pkgs.k8s.io [amd64]) for 'kubeadm'
    The following packages will be upgraded:
        kubeadm
    1 upgraded, 0 newly installed, 0 to remove and 17 not upgraded.
    Need to get 11.4 MB of archives.
    After this operation, 8,032 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.31/deb  kubeadm 1.31.4-1.1 [11.4 MB]
    Fetched 11.4 MB in 1s (14.8 MB/s)
    (Reading database ... 224773 files and directories currently installed.)
    Preparing to unpack .../kubeadm_1.31.4-1.1_amd64.deb ...
    Unpacking kubeadm (1.31.4-1.1) over (1.30.8-1.1) ...
    Setting up kubeadm (1.31.4-1.1) ...
    Scanning processes...                                                                           
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    No services need to be restarted.

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubeadm set on hold.
    ```

Verify and apply the upgrade plan:

??? terminal "`# kubeadm upgrade plan && kubeadm upgrade apply v1.31.4`"

    ``` console
    # kubeadm upgrade plan
    [preflight] Running pre-flight checks.
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [upgrade] Running cluster health checks
    W0110 22:45:52.938665 4182951 health.go:132] The preflight check "CreateJob" was skipped because there are no schedulable Nodes in the cluster.
    [upgrade] Fetching available versions to upgrade to
    [upgrade/versions] Cluster version: 1.30.8
    [upgrade/versions] kubeadm version: v1.31.4
    I0110 22:45:53.506618 4182951 version.go:261] remote version is much newer: v1.32.0; falling back to: stable-1.31
    [upgrade/versions] Target version: v1.31.4
    [upgrade/versions] Latest version in the v1.30 series: v1.30.8

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   NODE      CURRENT   TARGET
    kubelet     lexicon   v1.30.8   v1.31.4

    Upgrade to the latest stable version:

    COMPONENT                 NODE      CURRENT    TARGET
    kube-apiserver            lexicon   v1.30.8    v1.31.4
    kube-controller-manager   lexicon   v1.30.8    v1.31.4
    kube-scheduler            lexicon   v1.30.8    v1.31.4
    kube-proxy                          1.30.8     v1.31.4
    CoreDNS                             v1.11.3    v1.11.3
    etcd                      lexicon   3.5.16-0   3.5.15-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.31.4

    _____________________________________________________________________


    The table below shows the current state of component configs as understood by this version of kubeadm.
    Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
    resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
    upgrade to is denoted in the "PREFERRED VERSION" column.

    API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
    kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
    kubelet.config.k8s.io     v1beta1           v1beta1             no
    _____________________________________________________________________


    # kubeadm upgrade apply v1.31.4
    [preflight] Running pre-flight checks.
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [upgrade] Running cluster health checks
    W0110 22:45:59.615950 4184761 health.go:132] The preflight check "CreateJob" was skipped because there are no schedulable Nodes in the cluster.
    [upgrade/version] You have chosen to change the cluster version to "v1.31.4"
    [upgrade/versions] Cluster version: v1.30.8
    [upgrade/versions] kubeadm version: v1.31.4
    [upgrade] Are you sure you want to proceed? [y/N]: y
    [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
    [upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
    [upgrade/prepull] You can also perform this action beforehand using 'kubeadm config images pull'
    W0110 22:46:02.831598 4184761 checks.go:846] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "registry.k8s.io/pause:3.10" as the CRI sandbox image.
    [upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.31.4" (timeout: 5m0s)...
    [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests542531162"
    [upgrade/etcd] Non fatal issue encountered during upgrade: the desired etcd version "3.5.15-0" is older than the currently installed "3.5.16-0". Skipping etcd upgrade
    [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
    [upgrade/staticpods] Renewing apiserver certificate
    [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
    [upgrade/staticpods] Renewing front-proxy-client certificate
    [upgrade/staticpods] Renewing apiserver-etcd-client certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-22-46-13/kube-apiserver.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-apiserver
    [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
    [upgrade/staticpods] Renewing controller-manager.conf certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-22-46-13/kube-controller-manager.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-controller-manager
    [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
    [upgrade/staticpods] Renewing scheduler.conf certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-01-10-22-46-13/kube-scheduler.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-scheduler
    [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
    [upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config1002620026/config.yaml
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.31.4". Enjoy!

    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
    ```

Now that the control plane is updated, proceed to
[upgrade kubelet and kubectl](https://v1-29.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl)

??? terminal "`apt-get update && apt-get install -y kubelet='1.31.4-*' kubectl='1.31.4-*'`"

    ``` console
    # apt-mark unhold kubelet kubectl && \
    apt-get update && apt-get install -y kubelet='1.31.4-*' kubectl='1.31.4-*' && \
    apt-mark hold kubelet kubectl
    Canceled hold on kubelet.
    Canceled hold on kubectl.
    Ign:1 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 InRelease
    Hit:2 https://download.docker.com/linux/ubuntu jammy InRelease                                 
    Get:3 https://cli.github.com/packages stable InRelease [3,917 B]                               
    Hit:4 http://ch.archive.ubuntu.com/ubuntu jammy InRelease                                      
    Hit:5 http://ch.archive.ubuntu.com/ubuntu jammy-updates InRelease                              
    Hit:6 https://apt.releases.hashicorp.com jammy InRelease                                       
    Hit:7 https://apt.grafana.com stable InRelease                                                 
    Hit:9 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release                       
    Err:3 https://cli.github.com/packages stable InRelease                                         
        The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    Hit:8 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.31/deb  InRelease
    Hit:11 http://ch.archive.ubuntu.com/ubuntu jammy-backports InRelease                           
    Hit:10 https://dl.ui.com/unifi/debian unifi-7.3 InRelease           
    Hit:12 http://ch.archive.ubuntu.com/ubuntu jammy-security InRelease 
    Err:13 https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release.gpg
        The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    Hit:14 https://esm.ubuntu.com/apps/ubuntu jammy-apps-security InRelease
    Hit:15 https://esm.ubuntu.com/apps/ubuntu jammy-apps-updates InRelease
    Hit:16 https://esm.ubuntu.com/infra/ubuntu jammy-infra-security InRelease
    Hit:17 https://esm.ubuntu.com/infra/ubuntu jammy-infra-updates InRelease
    Reading package lists... Done
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://cli.github.com/packages stable InRelease: The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 Release: The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease  The following signatures were invalid: EXPKEYSIG 23F3D4EA75716059 GitHub CLI <opensource+cli@github.com>
    W: Failed to fetch https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/3.6/Release.gpg  The following signatures were invalid: EXPKEYSIG 58712A2291FA4AD5 MongoDB 3.6 Release Signing Key <packaging@mongodb.com>
    W: Some index files failed to download. They have been ignored, or old ones used instead.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.31.4-1.1' (isv:kubernetes:core:stable:v1.31:pkgs.k8s.io [amd64]) for 'kubelet'
    Selected version '1.31.4-1.1' (isv:kubernetes:core:stable:v1.31:pkgs.k8s.io [amd64]) for 'kubectl'
    The following packages will be upgraded:
        kubectl kubelet
    2 upgraded, 0 newly installed, 0 to remove and 15 not upgraded.
    Need to get 26.4 MB of archives.
    After this operation, 18.3 MB disk space will be freed.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.31/deb  kubectl 1.31.4-1.1 [11.2 MB]
    Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.31/deb  kubelet 1.31.4-1.1 [15.2 MB]
    Fetched 26.4 MB in 1s (36.9 MB/s)
    (Reading database ... 224773 files and directories currently installed.)
    Preparing to unpack .../kubectl_1.31.4-1.1_amd64.deb ...
    Unpacking kubectl (1.31.4-1.1) over (1.30.8-1.1) ...
    Preparing to unpack .../kubelet_1.31.4-1.1_amd64.deb ...
    Unpacking kubelet (1.31.4-1.1) over (1.30.8-1.1) ...
    Setting up kubectl (1.31.4-1.1) ...
    Setting up kubelet (1.31.4-1.1) ...
    Scanning processes...                                                                           
    Scanning candidates...                                                                          
    Scanning processor microcode...                                                                 
    Scanning linux images...                                                                        

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    Restarting services...
    systemctl restart kubelet.service

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    kubelet set on hold.
    kubectl set on hold.

    # systemctl daemon-reload
    # systemctl restart kubelet

    # kubectl  version --output=yaml
    clientVersion:
        buildDate: "2024-12-10T11:43:21Z"
        compiler: gc
        gitCommit: a78aa47129b8539636eb86a9d00e31b2720fe06b
        gitTreeState: clean
        gitVersion: v1.31.4
        goVersion: go1.22.9
        major: "1"
        minor: "31"
        platform: linux/amd64
    kustomizeVersion: v5.4.2
    serverVersion:
        buildDate: "2024-12-10T11:37:27Z"
        compiler: gc
        gitCommit: a78aa47129b8539636eb86a9d00e31b2720fe06b
        gitTreeState: clean
        gitVersion: v1.31.4
        goVersion: go1.22.9
        major: "1"
        minor: "31"
        platform: linux/amd64
    ```

Finally, bring the node back online by marking it schedulable:

??? terminal "`# kubectl uncordon lexicon`"

    ``` console
    # kubectl uncordon lexicon
    node/lexicon uncordoned
    # kubectl get pods -A
    NAMESPACE                NAME                                                     READY   STATUS              RESTARTS        AGE
    atuin-server             atuin-cdc5879b8-xkv7l                                    0/2     ContainerCreating   0               5m4s
    audiobookshelf           audiobookshelf-987b955fb-b7dsq                           0/1     ContainerCreating   0               5m4s
    cert-manager             cert-manager-64f9f45d6f-6zrtz                            0/1     ContainerCreating   0               5m4s
    cert-manager             cert-manager-cainjector-56bbdd5c47-h4w7c                 0/1     ContainerCreating   0               5m4s
    cert-manager             cert-manager-webhook-d4f4545d7-zvwr7                     0/1     ContainerCreating   0               5m4s
    code-server              code-server-5768c9d997-4mk8g                             0/1     ContainerCreating   0               5m4s
    default                  inteldeviceplugins-controller-manager-7994555cdb-87rsx   0/2     ContainerCreating   0               5m3s
    firefly-iii              firefly-iii-5bd9bb7c4-ps87g                              0/1     ContainerCreating   0               4m53s
    firefly-iii              firefly-iii-mysql-68f59d48f-lc9n5                        0/1     ContainerCreating   0               4m53s
    homebox                  homebox-7fb9d44d48-78s7v                                 0/1     ContainerCreating   0               5m4s
    ingress-nginx            ingress-nginx-controller-746f764ff4-lpvss                0/1     Running             0               5m4s
    kavita                   kavita-766577cf45-hr848                                  0/1     ContainerCreating   0               5m4s
    komga                    komga-549b6dd9b-xktm8                                    0/1     ContainerCreating   0               4m53s
    kube-flannel             kube-flannel-ds-nrrg6                                    1/1     Running             131 (16h ago)   650d
    kube-system              coredns-55cb58b774-dzhbn                                 0/1     ContainerCreating   0               4m53s
    kube-system              coredns-55cb58b774-n4fdw                                 0/1     ContainerCreating   0               4m53s
    kube-system              etcd-lexicon                                             1/1     Running             1 (27s ago)     23s
    kube-system              kube-apiserver-lexicon                                   1/1     Running             1 (27s ago)     23s
    kube-system              kube-controller-manager-lexicon                          1/1     Running             1 (27s ago)     23s
    kube-system              kube-proxy-59lmn                                         1/1     Running             1 (27s ago)     53s
    kube-system              kube-scheduler-lexicon                                   1/1     Running             1 (27s ago)     27s
    kubernetes-dashboard     dashboard-metrics-scraper-7bc864c59-khpwq                0/1     ContainerCreating   0               5m4s
    kubernetes-dashboard     kubernetes-dashboard-6c7ccbcf87-nzssm                    0/1     ContainerCreating   0               5m4s
    local-path-storage       local-path-provisioner-8bc8875b-f6s75                    0/1     ContainerCreating   0               4m53s
    metallb-system           controller-68bf958bf9-5rfgl                              0/1     ContainerCreating   0               4m53s
    metallb-system           speaker-t8f7k                                            1/1     Running             172 (27s ago)   649d
    metrics-server           metrics-server-74c749979-dpq84                           0/1     Running             0               4m53s
    monitoring               grafana-7647f97d64-wzgsp                                 0/1     ContainerCreating   0               4m53s
    monitoring               influxdb-57bddc4dc9-pqf8l                                1/1     Running             0               5m4s
    monitoring               telegraf-ztg2t                                           1/1     Running             40 (19s ago)    87d
    navidrome                navidrome-5f76b6ddf5-27sjc                               0/1     ContainerCreating   0               5m4s
    node-feature-discovery   nfd-node-feature-discovery-master-5f56c75d-d7xs5         0/1     ContainerCreating   0               4m53s
    node-feature-discovery   nfd-node-feature-discovery-worker-xmzwk                  1/1     Running             527 (27s ago)   483d
    plexserver               plexserver-5dbdfc898b-h7lm4                              0/1     ContainerCreating   0               4m53s
    unifi                    mongo-d55fbb7f5-p79nc                                    0/1     ContainerCreating   0               4m53s
    unifi                    unifi-6678ccc684-85kcr                                   0/1     ContainerCreating   0               5m4s
    ```

After a minute or two all the services are back up and running and the
Kubernetes dashboard shows all workloads green and OK again.

#### Kubernetes Certificate Check

[Kubernetes Certificate Expired](./2024-03-22-kubectl-certificate-expired.md)
less than a year ago, so it's about time to check whether the certificates
have been automatically renewed since then:

``` console
# ls -l /var/lib/kubelet/pki
total 32
-rw------- 1 root root 2826 Mar 22  2023 kubelet-client-2023-03-22-22-37-39.pem
-rw------- 1 root root 1110 Jan 15  2024 kubelet-client-2024-01-15-16-44-55.pem
-rw------- 1 root root 1110 Oct 11 06:08 kubelet-client-2024-10-11-06-08-33.pem
lrwxrwxrwx 1 root root   59 Oct 11 06:08 kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2024-10-11-06-08-33.pem
-rw-r--r-- 1 root root 2246 Mar 22  2023 kubelet.crt
-rw------- 1 root root 1675 Mar 22  2023 kubelet.key
-rw------- 1 root root 1147 Mar 23  2024 kubelet-server-2024-03-23-13-17-35.pem
lrwxrwxrwx 1 root root   59 Mar 23  2024 kubelet-server-current.pem -> /var/lib/kubelet/pki/kubelet-server-2024-03-23-13-17-35.pem

# openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -text -noout  | grep -A 2 Validity
        Validity
            Not Before: Mar 22 20:37:39 2023 GMT
            Not After : Mar 21 20:37:39 2024 GMT
```

*Nope*, the **server** certificate has not been renewed since then,
at least not under `/var/lib/kubelet/pki`, but at the same time
`kubeadm certs check-expiration` tells a different story: that
certificates **were** renewed today, as expectd, when upgrading:

``` console
# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jan 10, 2026 21:45 UTC   364d            ca                      no      
apiserver                  Jan 10, 2026 21:45 UTC   364d            ca                      no      
apiserver-etcd-client      Jan 10, 2026 21:45 UTC   364d            etcd-ca                 no      
apiserver-kubelet-client   Jan 10, 2026 21:45 UTC   364d            ca                      no      
controller-manager.conf    Jan 10, 2026 21:45 UTC   364d            ca                      no      
etcd-healthcheck-client    Jan 10, 2026 20:21 UTC   364d            etcd-ca                 no      
etcd-peer                  Jan 10, 2026 20:21 UTC   364d            etcd-ca                 no      
etcd-server                Jan 10, 2026 20:21 UTC   364d            etcd-ca                 no      
front-proxy-client         Jan 10, 2026 21:45 UTC   364d            front-proxy-ca          no      
scheduler.conf             Jan 10, 2026 21:45 UTC   364d            ca                      no      
super-admin.conf           Jan 10, 2026 21:45 UTC   364d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Mar 19, 2033 21:37 UTC   8y              no      
etcd-ca                 Mar 19, 2033 21:37 UTC   8y              no      
front-proxy-ca          Mar 19, 2033 21:37 UTC   8y              no      

# kubectl -n kube-system get cm kubeadm-config -o yaml
apiVersion: v1
data:
  ClusterConfiguration: |
    apiServer:
      extraArgs:
      - name: authorization-mode
        value: Node,RBAC
    apiVersion: kubeadm.k8s.io/v1beta4
    caCertificateValidityPeriod: 87600h0m0s
    certificateValidityPeriod: 8760h0m0s
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns: {}
    encryptionAlgorithm: RSA-2048
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: registry.k8s.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.31.4
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.244.0.0/16
      serviceSubnet: 10.96.0.0/12
    proxy: {}
    scheduler: {}
kind: ConfigMap
metadata:
  creationTimestamp: "2023-03-22T21:37:45Z"
  name: kubeadm-config
  namespace: kube-system
  resourceVersion: "90388726"
  uid: 0128b0c9-1380-442e-a8b1-4284cf1255bf
```

In the cluster configuration `certificatesDir: /etc/kubernetes/pki`, along with
[where certificates are stored](https://kubernetes.io/docs/setup/best-practices/certificates/#where-certificates-are-stored),
suggest that *those* are the relevant ceritifcats, not necessarily those under
`/var/lib/kubelet/pki/`, and indeed the former directory has renwed certs:

``` console
# ls -l /etc/kubernetes/pki/*
-rw-r--r-- 1 root root 1281 Jan 10 22:46 /etc/kubernetes/pki/apiserver.crt
-rw-r--r-- 1 root root 1123 Jan 10 22:46 /etc/kubernetes/pki/apiserver-etcd-client.crt
-rw------- 1 root root 1675 Jan 10 22:46 /etc/kubernetes/pki/apiserver-etcd-client.key
-rw------- 1 root root 1679 Jan 10 22:46 /etc/kubernetes/pki/apiserver.key
-rw-r--r-- 1 root root 1176 Jan 10 22:46 /etc/kubernetes/pki/apiserver-kubelet-client.crt
-rw------- 1 root root 1679 Jan 10 22:46 /etc/kubernetes/pki/apiserver-kubelet-client.key
-rw-r--r-- 1 root root 1099 Mar 22  2023 /etc/kubernetes/pki/ca.crt
-rw------- 1 root root 1679 Mar 22  2023 /etc/kubernetes/pki/ca.key
-rw-r--r-- 1 root root 1115 Mar 22  2023 /etc/kubernetes/pki/front-proxy-ca.crt
-rw------- 1 root root 1675 Mar 22  2023 /etc/kubernetes/pki/front-proxy-ca.key
-rw-r--r-- 1 root root 1119 Jan 10 22:46 /etc/kubernetes/pki/front-proxy-client.crt
-rw------- 1 root root 1679 Jan 10 22:46 /etc/kubernetes/pki/front-proxy-client.key
-rw------- 1 root root 1679 Mar 22  2023 /etc/kubernetes/pki/sa.key
-rw------- 1 root root  451 Mar 22  2023 /etc/kubernetes/pki/sa.pub

/etc/kubernetes/pki/etcd:
total 32
-rw-r--r-- 1 root root 1086 Mar 22  2023 ca.crt
-rw------- 1 root root 1675 Mar 22  2023 ca.key
-rw-r--r-- 1 root root 1123 Jan 10 21:21 healthcheck-client.crt
-rw------- 1 root root 1679 Jan 10 21:21 healthcheck-client.key
-rw-r--r-- 1 root root 1196 Jan 10 21:21 peer.crt
-rw------- 1 root root 1679 Jan 10 21:21 peer.key
-rw-r--r-- 1 root root 1196 Jan 10 21:21 server.crt
-rw------- 1 root root 1675 Jan 10 21:21 server.key
```

So it seems rotation of server certificates *is* working correctly.
