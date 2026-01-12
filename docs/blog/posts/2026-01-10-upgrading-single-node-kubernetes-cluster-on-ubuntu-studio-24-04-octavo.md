---
date: 2026-01-10
categories:
 - linux
 - ubuntu
 - server
 - kubernetes
 - docker
title: Upgrading single-node Kubernetes cluster on Ubuntu Studio 24.04 (octavo)
---

[Upgrading the single-node kubernetes cluster on `lexicon`](./2025-01-10-upgrading-single-node-kubernetes-cluster-on-ubuntu-studio-24-04-lexicon.md)
went smoothly, so it's time to repeat the process on `octavo` and `alfred`, *especially*
since the current version (1.32) will be the next one up to go
[End Of Life in Feb 28, 2026](https://kubernetes.io/releases/patch-releases/#non-active-branch-history/#1-32)
and a preemptive
[Kubernetes Certificate Check](./2025-01-10-upgrading-single-node-kubernetes-cluster-on-ubuntu-studio-24-04-lexicon.md#kubernetes-certificate-check)
reveals certificates are due to expire by Feb 22 on `alfred` and April 26 on `octavo`.

[Checking deployments before upgrading kubeadm clusters](2024-08-22-checking-deployments-before-upgrades.md)
again found results *mostly* reassuring for version 1.34.

<!-- more -->

## Upgrade to 1.33

The first upgrade would have required updating Ingress-NGINX to version 1.14 first, but
this requirement has been eliminated by replacing
[Ingress-NGINX with Pomerium](./2025-12-18-replacing-ingress-nginx-with-pomerium.md).

``` console
# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"32", GitVersion:"v1.32.4", GitCommit:"59526cd4867447956156ae3a602fcbac10a2c335", GitTreeState:"clean", BuildDate:"2025-04-22T16:02:27Z", GoVersion:"go1.23.6", Compiler:"gc", Platform:"linux/amd64"}

# kubectl get nodes
NAME     STATUS   ROLES           AGE    VERSION
octavo   Ready    control-plane   235d   v1.32.4
```

[Upgrade version 1.32.x to version 1.33.x](https://v1-33.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
starts by determining which version to upgrade to and updating the minor version in the repository configuration and then find the latest patch version, much like the previous
[Upgrade to 1.32](./2025-01-10-upgrading-single-node-kubernetes-cluster-on-ubuntu-studio-24-04-lexicon.md#upgrade-to-132):

``` console
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# vi /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /

# apt update
# apt-cache madison kubeadm
   kubeadm | 1.33.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
```

Now, **before** updating `kubeadm`, drain the node:

??? terminal "`# kubectl drain --ignore-daemonsets --delete-emptydir-data octavo`"

    ``` console hl_lines="2 48 87 91"
    # kubectl drain --ignore-daemonsets --delete-emptydir-data octavo
    Warning: ignoring DaemonSet-managed Pods: intel-device-plugins-gpu/intel-gpu-plugin-gpudeviceplugin-wpsqp, kube-flannel/kube-flannel-ds-m8h8n, kube-system/kube-proxy-zlhbr, metallb-system/speaker-92c2g, monitoring/prom-prometheus-node-exporter-r24zl, node-feature-discovery/node-feature-discovery-worker-jjglh
    evicting pod audiobookshelf/audiobookshelf-56749d45d4-bs484
    evicting pod code-server/code-server-85785b5848-t5qvz
    evicting pod komga/komga-5984f79858-c7tb7
    evicting pod cert-manager/cert-manager-webhook-78cb4cf989-wb4wz
    evicting pod cert-manager/cert-manager-cainjector-666b8b6b66-fl6rp
    evicting pod default/command-demo
    evicting pod cert-manager/cert-manager-7d67448f59-c2fgn
    evicting pod node-feature-discovery/node-feature-discovery-master-767dcc6cb8-rsrcc
    evicting pod firefly-iii/firefly-iii-f865768bf-f8tp6
    evicting pod pomerium/pomerium-gen-secrets-8sx7m
    evicting pod monitoring/prometheus-prom-kube-prometheus-stack-prometheus-0
    evicting pod metallb-system/controller-bb5f47665-vt57w
    evicting pod monitoring/prom-kube-prometheus-stack-operator-645fd684d6-n6qpf
    evicting pod navidrome/navidrome-75945786b-82mst
    evicting pod kubernetes-dashboard/kubernetes-dashboard-api-64c997cbcc-cxbjt
    evicting pod pomerium/pomerium-765bfd4df6-d8ft2
    evicting pod ryot/postgres-84f94497fd-nxt9v
    evicting pod home-assistant/home-assistant-5db584bd48-lf9v9
    evicting pod kubernetes-dashboard/kubernetes-dashboard-metrics-scraper-76df4956c4-bx6wj
    evicting pod monitoring/version-checker-547c9c998c-hmcff
    evicting pod default/ddns-updater-75d5df4647-dztn4
    evicting pod kube-system/coredns-668d6bf9bc-zpwqm
    evicting pod kube-system/coredns-668d6bf9bc-7fcdw
    evicting pod node-feature-discovery/node-feature-discovery-gc-5b65f7f5b6-jwpt4
    evicting pod kubernetes-dashboard/kubernetes-dashboard-web-56df7655d9-jc8ss
    evicting pod ryot/ryot-7d64cb5797-5kj65
    evicting pod unifi/mongo-7878d99ff5-q8lb6
    evicting pod firefly-iii/firefly-iii-mysql-659f959f57-jxqcz
    evicting pod homepage/homepage-58c79b7856-csdcm
    evicting pod monitoring/prom-kube-state-metrics-8576986c6b-xqcwl
    evicting pod tailscale/ts-home-assistant-tailscale-mdqlt-0
    evicting pod tailscale/operator-68b5df646d-c6tm9
    evicting pod kubernetes-dashboard/kubernetes-dashboard-kong-79867c9c48-dwncj
    evicting pod tailscale/ts-kubernetes-dashboard-ingress-tailscale-jhb6z-0
    evicting pod kubernetes-dashboard/kubernetes-dashboard-auth-5cf6848ffd-5vcm7
    evicting pod monitoring/grafana-695b647cb4-9ql86
    evicting pod monitoring/influxdb-76d6df578-z6w8n
    evicting pod monitoring/alertmanager-prom-kube-prometheus-stack-alertmanager-0
    evicting pod trivy-system/trivy-operator-59489786c6-hgg2b
    evicting pod monitoring/trivy-operator-dashboard-66d4c85cd6-rs2dq
    evicting pod unifi/unifi-6548ccd8c8-xkf24
    evicting pod media-center/jellyfin-6bbd7f9798-zvxq9
    evicting pod intel-device-plugins-gpu/inteldeviceplugins-controller-manager-7fb7c8d6b9-phnvq
    pod/pomerium-gen-secrets-8sx7m evicted
    pod/controller-bb5f47665-vt57w evicted
    I0110 13:15:04.930345 2878823 request.go:729] Waited for 1.000804357s due to client-side throttling, not priority and fairness, request: POST:https://10.0.0.8:6443/api/v1/namespaces/kubernetes-dashboard/pods/kubernetes-dashboard-metrics-scraper-76df4956c4-bx6wj/eviction
    pod/navidrome-75945786b-82mst evicted
    pod/prometheus-prom-kube-prometheus-stack-prometheus-0 evicted
    pod/pomerium-765bfd4df6-d8ft2 evicted
    pod/kubernetes-dashboard-api-64c997cbcc-cxbjt evicted
    pod/cert-manager-webhook-78cb4cf989-wb4wz evicted
    pod/prom-kube-prometheus-stack-operator-645fd684d6-n6qpf evicted
    pod/node-feature-discovery-master-767dcc6cb8-rsrcc evicted
    pod/komga-5984f79858-c7tb7 evicted
    pod/version-checker-547c9c998c-hmcff evicted
    pod/command-demo evicted
    pod/code-server-85785b5848-t5qvz evicted
    pod/cert-manager-cainjector-666b8b6b66-fl6rp evicted
    pod/cert-manager-7d67448f59-c2fgn evicted
    pod/kubernetes-dashboard-metrics-scraper-76df4956c4-bx6wj evicted
    pod/kubernetes-dashboard-web-56df7655d9-jc8ss evicted
    pod/postgres-84f94497fd-nxt9v evicted
    pod/node-feature-discovery-gc-5b65f7f5b6-jwpt4 evicted
    pod/ryot-7d64cb5797-5kj65 evicted
    pod/firefly-iii-mysql-659f959f57-jxqcz evicted
    pod/mongo-7878d99ff5-q8lb6 evicted
    pod/audiobookshelf-56749d45d4-bs484 evicted
    pod/prom-kube-state-metrics-8576986c6b-xqcwl evicted
    pod/homepage-58c79b7856-csdcm evicted
    pod/ts-home-assistant-tailscale-mdqlt-0 evicted
    pod/operator-68b5df646d-c6tm9 evicted
    pod/coredns-668d6bf9bc-zpwqm evicted
    pod/ddns-updater-75d5df4647-dztn4 evicted
    pod/coredns-668d6bf9bc-7fcdw evicted
    pod/kubernetes-dashboard-auth-5cf6848ffd-5vcm7 evicted
    pod/ts-kubernetes-dashboard-ingress-tailscale-jhb6z-0 evicted
    pod/influxdb-76d6df578-z6w8n evicted
    pod/home-assistant-5db584bd48-lf9v9 evicted
    pod/grafana-695b647cb4-9ql86 evicted
    pod/trivy-operator-59489786c6-hgg2b evicted
    pod/trivy-operator-dashboard-66d4c85cd6-rs2dq evicted
    pod/alertmanager-prom-kube-prometheus-stack-alertmanager-0 evicted
    pod/jellyfin-6bbd7f9798-zvxq9 evicted
    pod/inteldeviceplugins-controller-manager-7fb7c8d6b9-phnvq evicted
    I0110 13:15:15.098024 2878823 request.go:729] Waited for 2.19577063s due to client-side throttling, not priority and fairness, request: GET:https://10.0.0.8:6443/api/v1/namespaces/firefly-iii/pods/firefly-iii-f865768bf-f8tp6
    pod/kubernetes-dashboard-kong-79867c9c48-dwncj evicted
    pod/unifi-6548ccd8c8-xkf24 evicted
    pod/firefly-iii-f865768bf-f8tp6 evicted
    node/octavo drained
    ```

The latest patch version is 1.33.7 and that is the one to [upgrade control plane nodes](https://v1-33.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes):

??? terminal "`# apt-get update && apt-get install -y kubeadm='1.33.7-*' && apt-mark hold kubeadm`"

    ``` console
    # apt-mark unhold kubeadm && \
      apt-get update && apt-get install -y kubeadm='1.33.7-*' && \
      apt-mark hold kubeadm
    Canceled hold on kubeadm.
    Fetched 6,581 B in 1s (7,931 B/s)
    Reading package lists... Done
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.33.7-1.1' (isv:kubernetes:core:stable:v1.33:pkgs.k8s.io [amd64]) for 'kubeadm'
    The following packages will be upgraded:
      kubeadm
    1 upgraded, 0 newly installed, 0 to remove and 4 not upgraded.
    Need to get 12.7 MB of archives.
    After this operation, 3,609 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.33/deb  kubeadm 1.33.7-1.1 [12.7 MB]
    Fetched 12.7 MB in 0s (45.6 MB/s)
    (Reading database ... 168496 files and directories currently installed.)
    Preparing to unpack .../kubeadm_1.33.7-1.1_amd64.deb ...
    Unpacking kubeadm (1.33.7-1.1) over (1.32.4-1.1) ...
    Setting up kubeadm (1.33.7-1.1) ...
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

??? terminal "`# kubeadm upgrade plan`"

    ``` console
    # kubeadm upgrade plan
    [preflight] Running pre-flight checks.
    [upgrade/config] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
    [upgrade/config] Use 'kubeadm init phase upload-config --config your-config-file' to re-upload it.
    [upgrade] Running cluster health checks
    W0110 13:20:13.836378 2975827 health.go:135] The preflight check "CreateJob" was skipped because there are no schedulable Nodes in the cluster.
    [upgrade] Fetching available versions to upgrade to
    [upgrade/versions] Cluster version: 1.32.4
    [upgrade/versions] kubeadm version: v1.33.7
    I0110 13:20:14.375889 2975827 version.go:261] remote version is much newer: v1.35.0; falling back to: stable-1.33
    [upgrade/versions] Target version: v1.33.7
    [upgrade/versions] Latest version in the v1.32 series: v1.32.11

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   NODE      CURRENT   TARGET
    kubelet     octavo    v1.32.4   v1.32.11

    Upgrade to the latest version in the v1.32 series:

    COMPONENT                 NODE      CURRENT    TARGET
    kube-apiserver            octavo    v1.32.4    v1.32.11
    kube-controller-manager   octavo    v1.32.4    v1.32.11
    kube-scheduler            octavo    v1.32.4    v1.32.11
    kube-proxy                          1.32.4     v1.32.11
    CoreDNS                             v1.11.3    v1.12.0
    etcd                      octavo    3.5.16-0   3.5.24-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.32.11

    _____________________________________________________________________

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   NODE      CURRENT   TARGET
    kubelet     octavo    v1.32.4   v1.33.7

    Upgrade to the latest stable version:

    COMPONENT                 NODE      CURRENT    TARGET
    kube-apiserver            octavo    v1.32.4    v1.33.7
    kube-controller-manager   octavo    v1.32.4    v1.33.7
    kube-scheduler            octavo    v1.32.4    v1.33.7
    kube-proxy                          1.32.4     v1.33.7
    CoreDNS                             v1.11.3    v1.12.0
    etcd                      octavo    3.5.16-0   3.5.24-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.33.7

    _____________________________________________________________________


    The table below shows the current state of component configs as understood by this version of kubeadm.
    Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
    resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
    upgrade to is denoted in the "PREFERRED VERSION" column.

    API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
    kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
    kubelet.config.k8s.io     v1beta1           v1beta1             no
    _____________________________________________________________________

    ```

??? terminal "`# kubeadm upgrade apply v1.33.7`"

    ``` console
    # kubeadm upgrade apply v1.33.7
    [upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
    [upgrade] Use 'kubeadm init phase upload-config --config your-config-file' to re-upload it.
    [upgrade/preflight] Running preflight checks
    [upgrade] Running cluster health checks
    W0110 13:21:59.695587 3005821 health.go:135] The preflight check "CreateJob" was skipped because there are no schedulable Nodes in the cluster.
    [upgrade/preflight] You have chosen to upgrade the cluster version to "v1.33.7"
    [upgrade/versions] Cluster version: v1.32.4
    [upgrade/versions] kubeadm version: v1.33.7
    [upgrade] Are you sure you want to proceed? [y/N]: y
    [upgrade/preflight] Pulling images required for setting up a Kubernetes cluster
    [upgrade/preflight] This might take a minute or two, depending on the speed of your internet connection
    [upgrade/preflight] You can also perform this action beforehand using 'kubeadm config images pull'
    [upgrade/control-plane] Upgrading your static Pod-hosted control plane to version "v1.33.7" (timeout: 5m0s)...
    [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests1594195257"
    [upgrade/staticpods] Preparing for "etcd" upgrade
    [upgrade/staticpods] Renewing etcd-server certificate
    [upgrade/staticpods] Renewing etcd-peer certificate
    [upgrade/staticpods] Renewing etcd-healthcheck-client certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-01-10-13-22-17/etcd.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=etcd
    [upgrade/staticpods] Component "etcd" upgraded successfully!
    [upgrade/etcd] Waiting for etcd to become available
    [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
    [upgrade/staticpods] Renewing apiserver certificate
    [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
    [upgrade/staticpods] Renewing front-proxy-client certificate
    [upgrade/staticpods] Renewing apiserver-etcd-client certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-01-10-13-22-17/kube-apiserver.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-apiserver
    [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
    [upgrade/staticpods] Renewing controller-manager.conf certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-01-10-13-22-17/kube-controller-manager.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-controller-manager
    [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
    [upgrade/staticpods] Renewing scheduler.conf certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-01-10-13-22-17/kube-scheduler.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-scheduler
    [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
    [upgrade/control-plane] The control plane instance for this node was successfully upgraded!
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
    [upgrade/kubeconfig] The kubeconfig files for this node were successfully upgraded!
    W0110 13:25:05.926913 3005821 postupgrade.go:117] Using temporary directory /etc/kubernetes/tmp/kubeadm-kubelet-config1025000842 for kubelet config. To override it set the environment variable KUBEADM_UPGRADE_DRYRUN_DIR
    [upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config1025000842/config.yaml
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
    [upgrade/bootstrap-token] Configuring bootstrap token and cluster-info RBAC rules
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    [upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.33.7".

    [upgrade] Now please proceed with upgrading the rest of the nodes by following the right order.
    ```

Now that the control plane is updated, proceed to 
[upgrade kubelet and kubectl](https://v1-33.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl):

??? terminal "`# apt-mark unhold kubelet kubectl && apt-get update && apt-get install -y kubelet='1.33.7-*' kubectl='1.33.7-*' && apt-mark hold kubelet kubectl`"

    ``` console
    # apt-mark unhold kubelet kubectl && \
    apt-get update && apt-get install -y kubelet='1.33.7-*' kubectl='1.33.7-*' && \
    apt-mark hold kubelet kubectl
    Canceled hold on kubelet.
    Canceled hold on kubectl.
    Fetched 6,581 B in 1s (5,712 B/s)
    Reading package lists... Done
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.33.7-1.1' (isv:kubernetes:core:stable:v1.33:pkgs.k8s.io [amd64]) for 'kubelet'
    Selected version '1.33.7-1.1' (isv:kubernetes:core:stable:v1.33:pkgs.k8s.io [amd64]) for 'kubectl'
    The following package was automatically installed and is no longer required:
      conntrack
    Use 'apt autoremove' to remove it.
    The following packages will be upgraded:
      kubectl kubelet
    2 upgraded, 0 newly installed, 0 to remove and 2 not upgraded.
    Need to get 27.6 MB of archives.
    After this operation, 7,123 kB of additional disk space will be used.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.33/deb  kubectl 1.33.7-1.1 [11.7 MB]
    Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.33/deb  kubelet 1.33.7-1.1 [15.9 MB]
    Fetched 27.6 MB in 0s (72.5 MB/s) 
    (Reading database ... 168496 files and directories currently installed.)
    Preparing to unpack .../kubectl_1.33.7-1.1_amd64.deb ...
    Unpacking kubectl (1.33.7-1.1) over (1.32.4-1.1) ...
    Preparing to unpack .../kubelet_1.33.7-1.1_amd64.deb ...
    Unpacking kubelet (1.33.7-1.1) over (1.32.4-1.1) ...
    Setting up kubectl (1.33.7-1.1) ...
    Setting up kubelet (1.33.7-1.1) ...
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
    ```

At this point `kubectl` should confirm the cluster is fully upgraded:

``` console
# kubectl  version --output=yaml
clientVersion:
  buildDate: "2025-12-09T14:42:24Z"
  compiler: gc
  gitCommit: a7245cdf3f69e11356c7e8f92b3e78ca4ee4e757
  gitTreeState: clean
  gitVersion: v1.33.7
  goVersion: go1.24.11
  major: "1"
  minor: "33"
  platform: linux/amd64
kustomizeVersion: v5.6.0
serverVersion:
  buildDate: "2025-12-09T14:35:23Z"
  compiler: gc
  emulationMajor: "1"
  emulationMinor: "33"
  gitCommit: a7245cdf3f69e11356c7e8f92b3e78ca4ee4e757
  gitTreeState: clean
  gitVersion: v1.33.7
  goVersion: go1.24.11
  major: "1"
  minCompatibilityMajor: "1"
  minCompatibilityMinor: "32"
  minor: "33"
  platform: linux/amd64
```

Finally, bring the node back online by marking it schedulable:

``` console
# kubectl uncordon octavo
node/octavo uncordoned
```

After a couple of minutes all services are back up.

## Update Flannel before 1.34

Upgrading networking components in Kubernetes requires careful sequencing. Flannel
v0.27.0 is highly optimized for the 1.34 release cycle, and while Kubernetes 1.34
is designed for minimal disruption, the following risks apply to your upgrade strategies.

Updating the CNI (Flannel) while still on Kubernetes 1.33 is generally considered the safest approach.

+   Risks:
    +   Configuration Drift: New default settings in v0.27.0 might not fully align with
        older 1.33 kube-proxy behaviors, though this is rare for minor version bumps.
    *   DaemonSet Restart: Updating Flannel requires restarting its pods across all
        nodes. If the update fails (e.g., image pull error), you risk losing pod-to-pod
        connectivity across the entire cluster.
+    Benefits: Ensures the networking layer is ready for the new APIs and kernel optimizations that Kubernetes 1.34 utilizes.

Recommended Upgrade Order:

1.  Backup your cluster configuration and etcd state.
1.  Upgrade Flannel to v0.27.x while on Kubernetes 1.33. Monitor for stability for 24 hours.
1.  Upgrade the Control Plane (API Server, Controller Manager) to Kubernetes 1.34.
1.  Upgrade Worker Nodes sequentially (Cordon, Drain, Upgrade Kubelet) to Kubernetes 1.34. 

Before upgrading Flannel, make a backup of its `configmap`:

``` console
$ kubectl get configmap -n kube-flannel kube-flannel-cfg -o yaml \
  > flannel-config-backup.yaml
```

Then update Flannel by re-downloading its latest release of `kube-flannel.yml`

``` console
$ wget -O kube-flannel.yml  \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

This is necessary because updating Flannel often involves more than just increasing its
version numbers, as this `git diff` shows comparing the latest release from the one
nearly a year ago:

``` console hl_lines="10-13 21-22 30-31"
$ git diff kube-flannel.yml
diff --git a/kube-flannel.yml b/kube-flannel.yml
index 3576fa3..9f16884 100644
--- a/kube-flannel.yml
+++ b/kube-flannel.yml
@@ -143,7 +143,9 @@ spec:
               fieldPath: metadata.namespace
         - name: EVENT_QUEUE_DEPTH
           value: "5000"
-        image: ghcr.io/flannel-io/flannel:v0.26.7
+        - name: CONT_WHEN_CACHE_NOT_READY
+          value: "false"
+        image: ghcr.io/flannel-io/flannel:v0.28.0
         name: kube-flannel
         resources:
           requests:
@@ -170,7 +172,7 @@ spec:
         - /opt/cni/bin/flannel
         command:
         - cp
-        image: ghcr.io/flannel-io/flannel-cni-plugin:v1.6.2-flannel1
+        image: ghcr.io/flannel-io/flannel-cni-plugin:v1.8.0-flannel1
         name: install-cni-plugin
         volumeMounts:
         - mountPath: /opt/cni/bin
@@ -181,7 +183,7 @@ spec:
         - /etc/cni/net.d/10-flannel.conflist
         command:
         - cp
-        image: ghcr.io/flannel-io/flannel:v0.26.7
+        image: ghcr.io/flannel-io/flannel:v0.28.0
         name: install-cni
         volumeMounts:
         - mountPath: /etc/cni/net.d
``` 

The update itself is then as simpley as (re)applying the manifest:

``` console
$ kubectl apply -f kube-flannel.yml
namespace/kube-flannel unchanged
serviceaccount/flannel unchanged
clusterrole.rbac.authorization.k8s.io/flannel unchanged
clusterrolebinding.rbac.authorization.k8s.io/flannel unchanged
configmap/kube-flannel-cfg unchanged
daemonset.apps/kube-flannel-ds configured
```

## Upgrade to 1.34

*Mostly* the same as the [upgrade to 1.33](#upgrade-to-133) with an important caveat.

[Upgrade version 1.33.x to version 1.34.x](https://v1-34.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
starts by determining which version to upgrade to and updating the minor version in the repository configuration and then find the latest patch version, much like the previous
[Upgrade to 1.33](#upgrade-to-133):

``` console
# curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# vi /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /

# apt update
# apt-cache madison kubeadm
   kubeadm | 1.34.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
```

Again, **before** updating `kubeadm`, drain the node:

??? terminal "`# kubectl drain --ignore-daemonsets --delete-emptydir-data octavo`"

    ``` console
    # kubectl drain --ignore-daemonsets --delete-emptydir-data octavo
    node/octavo cordoned
    Warning: ignoring DaemonSet-managed Pods: intel-device-plugins-gpu/intel-gpu-plugin-gpudeviceplugin-wpsqp, kube-flannel/kube-flannel-ds-ctb92, kube-system/kube-proxy-w725k, metallb-system/speaker-92c2g, monitoring/prom-prometheus-node-exporter-r24zl, node-feature-discovery/node-feature-discovery-worker-jjglh
    evicting pod cert-manager/cert-manager-cainjector-666b8b6b66-85xn6
    evicting pod code-server/code-server-85785b5848-hfbpf
    evicting pod unifi/unifi-6548ccd8c8-pjvcc
    evicting pod cert-manager/cert-manager-7d67448f59-sww8j
    evicting pod monitoring/influxdb-76d6df578-vdqck
    evicting pod kubernetes-dashboard/kubernetes-dashboard-api-64c997cbcc-lk2xt
    evicting pod node-feature-discovery/node-feature-discovery-master-767dcc6cb8-jg4sz
    evicting pod monitoring/prom-kube-prometheus-stack-operator-645fd684d6-w75qd
    evicting pod cert-manager/cert-manager-webhook-78cb4cf989-9zktk
    evicting pod metallb-system/controller-bb5f47665-mhtlg
    evicting pod homepage/homepage-58c79b7856-ldwrc
    evicting pod navidrome/navidrome-75945786b-6mj66
    evicting pod monitoring/alertmanager-prom-kube-prometheus-stack-alertmanager-0
    evicting pod media-center/jellyfin-6bbd7f9798-k4hj4
    evicting pod kubernetes-dashboard/kubernetes-dashboard-web-56df7655d9-z9bp2
    evicting pod monitoring/trivy-operator-dashboard-66d4c85cd6-fmz7d
    evicting pod kube-system/coredns-674b8bbfcf-ss8wf
    evicting pod tailscale/operator-68b5df646d-zs465
    evicting pod intel-device-plugins-gpu/inteldeviceplugins-controller-manager-7fb7c8d6b9-gfjwh
    evicting pod node-feature-discovery/node-feature-discovery-gc-5b65f7f5b6-ld7mh
    evicting pod firefly-iii/firefly-iii-mysql-659f959f57-6hr8p
    evicting pod ryot/postgres-84f94497fd-ql4pf
    evicting pod default/ddns-updater-75d5df4647-rxfzw
    evicting pod komga/komga-5984f79858-fzw5l
    evicting pod monitoring/version-checker-547c9c998c-hvv7l
    evicting pod ryot/ryot-848d9f4b6b-zqmfx
    evicting pod monitoring/prom-kube-state-metrics-8576986c6b-9wvfh
    evicting pod home-assistant/home-assistant-5db584bd48-c4xwz
    evicting pod kube-system/coredns-674b8bbfcf-9j8qs
    evicting pod tailscale/ts-kubernetes-dashboard-ingress-tailscale-jhb6z-0
    evicting pod kubernetes-dashboard/kubernetes-dashboard-auth-5cf6848ffd-r2zmq
    evicting pod kubernetes-dashboard/kubernetes-dashboard-kong-79867c9c48-5h28m
    evicting pod kubernetes-dashboard/kubernetes-dashboard-metrics-scraper-76df4956c4-5bkdg
    evicting pod tailscale/ts-home-assistant-tailscale-mdqlt-0
    evicting pod firefly-iii/firefly-iii-f865768bf-vhqk6
    evicting pod monitoring/prometheus-prom-kube-prometheus-stack-prometheus-0
    evicting pod trivy-system/trivy-operator-59489786c6-dzlqj
    evicting pod audiobookshelf/audiobookshelf-5798dfc5d5-wm6ts
    evicting pod unifi/mongo-7878d99ff5-6q6z6
    evicting pod monitoring/grafana-695b647cb4-jk7kr
    evicting pod pomerium/pomerium-765bfd4df6-5nn2r
    I0111 22:55:16.851803 1528671 request.go:752] "Waited before sending request" delay="1.000274541s" reason="client-side throttling, not priority and fairness" verb="POST" URL="https://10.0.0.8:6443/api/v1/namespaces/trivy-system/pods/trivy-operator-59489786c6-dzlqj/eviction"
    pod/homepage-58c79b7856-ldwrc evicted
    pod/cert-manager-cainjector-666b8b6b66-85xn6 evicted
    pod/ts-home-assistant-tailscale-mdqlt-0 evicted
    pod/prometheus-prom-kube-prometheus-stack-prometheus-0 evicted
    pod/kubernetes-dashboard-auth-5cf6848ffd-r2zmq evicted
    pod/prom-kube-state-metrics-8576986c6b-9wvfh evicted
    pod/kubernetes-dashboard-api-64c997cbcc-lk2xt evicted
    pod/audiobookshelf-5798dfc5d5-wm6ts evicted
    pod/ryot-848d9f4b6b-zqmfx evicted
    pod/version-checker-547c9c998c-hvv7l evicted
    pod/inteldeviceplugins-controller-manager-7fb7c8d6b9-gfjwh evicted
    pod/postgres-84f94497fd-ql4pf evicted
    pod/ts-kubernetes-dashboard-ingress-tailscale-jhb6z-0 evicted
    pod/jellyfin-6bbd7f9798-k4hj4 evicted
    pod/alertmanager-prom-kube-prometheus-stack-alertmanager-0 evicted
    pod/trivy-operator-59489786c6-dzlqj evicted
    pod/komga-5984f79858-fzw5l evicted
    pod/code-server-85785b5848-hfbpf evicted
    pod/node-feature-discovery-master-767dcc6cb8-jg4sz evicted
    pod/prom-kube-prometheus-stack-operator-645fd684d6-w75qd evicted
    pod/cert-manager-7d67448f59-sww8j evicted
    pod/cert-manager-webhook-78cb4cf989-9zktk evicted
    pod/trivy-operator-dashboard-66d4c85cd6-fmz7d evicted
    pod/coredns-674b8bbfcf-9j8qs evicted
    pod/influxdb-76d6df578-vdqck evicted
    pod/kubernetes-dashboard-web-56df7655d9-z9bp2 evicted
    pod/controller-bb5f47665-mhtlg evicted
    pod/node-feature-discovery-gc-5b65f7f5b6-ld7mh evicted
    pod/ddns-updater-75d5df4647-rxfzw evicted
    pod/unifi-6548ccd8c8-pjvcc evicted
    pod/operator-68b5df646d-zs465 evicted
    pod/navidrome-75945786b-6mj66 evicted
    pod/kubernetes-dashboard-metrics-scraper-76df4956c4-5bkdg evicted
    pod/firefly-iii-mysql-659f959f57-6hr8p evicted
    pod/grafana-695b647cb4-jk7kr evicted
    pod/pomerium-765bfd4df6-5nn2r evicted
    pod/mongo-7878d99ff5-6q6z6 evicted
    pod/coredns-674b8bbfcf-ss8wf evicted
    pod/home-assistant-5db584bd48-c4xwz evicted
    pod/kubernetes-dashboard-kong-79867c9c48-5h28m evicted
    pod/firefly-iii-f865768bf-vhqk6 evicted
    node/octavo drained
    ```

Again, **before** updating `kubeadm`, drain the node:

??? terminal "`# apt-get update && apt-get install kubeadm='1.34.3-*' && apt-mark hold kubeadm`"

    ``` console
    # apt-get update && \
      apt-get install kubeadm='1.34.3-*' && \
      apt-mark hold kubeadm
    Fetched 6,581 B in 1s (7,701 B/s)
    Reading package lists... Done
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.34.3-1.1' (isv:kubernetes:core:stable:v1.34:pkgs.k8s.io [amd64]) for 'kubeadm'
    The following package was automatically installed and is no longer required:
      conntrack
    Use 'apt autoremove' to remove it.
    The following held packages will be changed:
      kubeadm
    The following packages will be upgraded:
      kubeadm
    1 upgraded, 0 newly installed, 0 to remove and 4 not upgraded.
    Need to get 12.5 MB of archives.
    After this operation, 524 kB disk space will be freed.
    Do you want to continue? [Y/n]  
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.34/deb  kubeadm 1.34.3-1.1 [12.5 MB]
    Fetched 12.5 MB in 1s (23.5 MB/s)
    (Reading database ... 168512 files and directories currently installed.)
    Preparing to unpack .../kubeadm_1.34.3-1.1_amd64.deb ...
    Unpacking kubeadm (1.34.3-1.1) over (1.33.7-1.1) ...
    Setting up kubeadm (1.34.3-1.1) ...
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

??? terminal "`# kubeadm upgrade plan`"

    ``` console
    # kubeadm upgrade plan
    [preflight] Running pre-flight checks.
    [upgrade/config] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
    [upgrade/config] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
    [upgrade] Running cluster health checks
    W0111 22:56:39.632491 1563261 health.go:134] The preflight check "CreateJob" was skipped because there are no schedulable Nodes in the cluster.
    [upgrade] Fetching available versions to upgrade to
    [upgrade/versions] Cluster version: 1.33.7
    [upgrade/versions] kubeadm version: v1.34.3
    I0111 22:56:40.075055 1563261 version.go:260] remote version is much newer: v1.35.0; falling back to: stable-1.34
    [upgrade/versions] Target version: v1.34.3
    [upgrade/versions] Latest version in the v1.33 series: v1.33.7

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   NODE      CURRENT   TARGET
    kubelet     octavo    v1.33.7   v1.34.3

    Upgrade to the latest stable version:

    COMPONENT                 NODE      CURRENT    TARGET
    kube-apiserver            octavo    v1.33.7    v1.34.3
    kube-controller-manager   octavo    v1.33.7    v1.34.3
    kube-scheduler            octavo    v1.33.7    v1.34.3
    kube-proxy                          1.33.7     v1.34.3
    CoreDNS                             v1.12.0    v1.12.1
    etcd                      octavo    3.5.24-0   3.6.5-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.34.3

    _____________________________________________________________________


    The table below shows the current state of component configs as understood by this version of kubeadm.
    Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
    resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
    upgrade to is denoted in the "PREFERRED VERSION" column.

    API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
    kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
    kubelet.config.k8s.io     v1beta1           v1beta1             no
    _____________________________________________________________________
    ```

??? terminal "`# kubeadm upgrade apply v1.33.7`"

    ``` console
    # kubeadm upgrade apply v1.34.3
    [upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
    [upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
    [upgrade/preflight] Running preflight checks
    [upgrade] Running cluster health checks
    W0111 22:56:49.519658 1565941 health.go:134] The preflight check "CreateJob" was skipped because there are no schedulable Nodes in the cluster.
    [upgrade/preflight] You have chosen to upgrade the cluster version to "v1.34.3"
    [upgrade/versions] Cluster version: v1.33.7
    [upgrade/versions] kubeadm version: v1.34.3
    [upgrade] Are you sure you want to proceed? [y/N]: y
    [upgrade/preflight] Pulling images required for setting up a Kubernetes cluster
    [upgrade/preflight] This might take a minute or two, depending on the speed of your internet connection
    [upgrade/preflight] You can also perform this action beforehand using 'kubeadm config images pull'
    [upgrade/control-plane] Upgrading your static Pod-hosted control plane to version "v1.34.3" (timeout: 5m0s)...
    [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests1221527384"
    [upgrade/staticpods] Preparing for "etcd" upgrade
    [upgrade/staticpods] Renewing etcd-server certificate
    [upgrade/staticpods] Renewing etcd-peer certificate
    [upgrade/staticpods] Renewing etcd-healthcheck-client certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-01-11-22-57-01/etcd.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=etcd
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [Get "https://10.0.0.8:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd": dial tcp 10.0.0.8:6443: connect: connection refused]
    [apiclient] Error getting Pods with label selector "component=etcd" [pods is forbidden: User "kubernetes-admin" cannot list resource "pods" in API group "" in the namespace "kube-system"]
    [upgrade/staticpods] Component "etcd" upgraded successfully!
    [upgrade/etcd] Waiting for etcd to become available
    [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
    [upgrade/staticpods] Renewing apiserver certificate
    [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
    [upgrade/staticpods] Renewing front-proxy-client certificate
    [upgrade/staticpods] Renewing apiserver-etcd-client certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-01-11-22-57-01/kube-apiserver.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-apiserver
    [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
    [upgrade/staticpods] Renewing controller-manager.conf certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-01-11-22-57-01/kube-controller-manager.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-controller-manager
    [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
    [upgrade/staticpods] Renewing scheduler.conf certificate
    [upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-01-11-22-57-01/kube-scheduler.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This can take up to 5m0s
    [apiclient] Found 1 Pods for label selector component=kube-scheduler
    [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
    [upgrade/control-plane] The control plane instance for this node was successfully upgraded!
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
    [upgrade/kubeconfig] The kubeconfig files for this node were successfully upgraded!
    W0111 23:00:28.196018 1565941 postupgrade.go:116] Using temporary directory /etc/kubernetes/tmp/kubeadm-kubelet-config1928247334 for kubelet config. To override it set the environment variable KUBEADM_UPGRADE_DRYRUN_DIR
    [upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config1928247334/config.yaml
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
    [patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
    [upgrade/bootstrap-token] Configuring bootstrap token and cluster-info RBAC rules
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    [upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.34.3".

    [upgrade] Now please proceed with upgrading the rest of the nodes by following the right order.
    ```

Now that the control plane is updated, proceed to 
[upgrade kubelet and kubectl](https://v1-34.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl):

??? terminal "`# apt-mark unhold kubelet kubectl && apt-get update && apt-get install -y kubelet='1.33.7-*' kubectl='1.33.7-*' && apt-mark hold kubelet kubectl`"

    ``` console
    # apt-mark unhold kubelet kubectl && \
      apt-get update && \
      apt-get install kubelet='1.34.3-*' kubectl='1.34.3-*' && \
      apt-mark hold kubelet kubectl
    Canceled hold on kubelet.
    Canceled hold on kubectl.
    Fetched 6,581 B in 1s (5,888 B/s)
    Reading package lists... Done
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Selected version '1.34.3-1.1' (isv:kubernetes:core:stable:v1.34:pkgs.k8s.io [amd64]) for 'kubelet'
    Selected version '1.34.3-1.1' (isv:kubernetes:core:stable:v1.34:pkgs.k8s.io [amd64]) for 'kubectl'
    The following package was automatically installed and is no longer required:
      conntrack
    Use 'apt autoremove' to remove it.
    The following packages will be upgraded:
      kubectl kubelet
    2 upgraded, 0 newly installed, 0 to remove and 2 not upgraded.
    Need to get 24.7 MB of archives.
    After this operation, 22.1 MB disk space will be freed.
    Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.34/deb  kubectl 1.34.3-1.1 [11.7 MB]
    Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.34/deb  kubelet 1.34.3-1.1 [13.0 MB]
    Fetched 24.7 MB in 1s (40.2 MB/s) 
    (Reading database ... 168512 files and directories currently installed.)
    Preparing to unpack .../kubectl_1.34.3-1.1_amd64.deb ...
    Unpacking kubectl (1.34.3-1.1) over (1.33.7-1.1) ...
    Preparing to unpack .../kubelet_1.34.3-1.1_amd64.deb ...
    Unpacking kubelet (1.34.3-1.1) over (1.33.7-1.1) ...
    Setting up kubectl (1.34.3-1.1) ...
    Setting up kubelet (1.34.3-1.1) ...
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
    ```

At this point `kubectl` should confirm the cluster is fully upgraded:

``` console
# kubectl  version --output=yaml
clientVersion:
  buildDate: "2025-12-09T15:06:39Z"
  compiler: gc
  gitCommit: df11db1c0f08fab3c0baee1e5ce6efbf816af7f1
  gitTreeState: clean
  gitVersion: v1.34.3
  goVersion: go1.24.11
  major: "1"
  minor: "34"
  platform: linux/amd64
kustomizeVersion: v5.7.1
serverVersion:
  buildDate: "2025-12-09T14:59:13Z"
  compiler: gc
  emulationMajor: "1"
  emulationMinor: "34"
  gitCommit: df11db1c0f08fab3c0baee1e5ce6efbf816af7f1
  gitTreeState: clean
  gitVersion: v1.34.3
  goVersion: go1.24.11
  major: "1"
  minCompatibilityMajor: "1"
  minCompatibilityMinor: "33"
  minor: "34"
  platform: linux/amd64
```

Finally, bring the node back online by marking it schedulable:

``` console
# kubectl uncordon octavo
node/octavo uncordoned
```

After a couple of minutes *most* services are back up...

### Fix Firefly-III

Once all servicers *should* be back up, it is always good to check that they **are**
*actually* back up.

This time, just one was not yet ready:


``` console hl_lines="10 22 35"
$ kubectl get all -A
NAMESPACE                  NAME                                                         READY   STATUS             RESTARTS         AGE
audiobookshelf             pod/audiobookshelf-5798dfc5d5-nhrdm                          1/1     Running            0                8m8s
cert-manager               pod/cert-manager-7d67448f59-5llzl                            1/1     Running            0                7m56s
cert-manager               pod/cert-manager-cainjector-666b8b6b66-x4dck                 1/1     Running            0                8m8s
cert-manager               pod/cert-manager-webhook-78cb4cf989-kb24w                    1/1     Running            0                7m56s
code-server                pod/code-server-85785b5848-9ln8r                             1/1     Running            0                8m7s
default                    pod/ddns-updater-75d5df4647-bnstp                            1/1     Running            0                7m56s
firefly-iii                pod/firefly-iii-f865768bf-kgjq7                              1/1     Running            0                8m7s
firefly-iii                pod/firefly-iii-mysql-659f959f57-ph92z                       0/1     ImagePullBackOff   0                7m56s
home-assistant             pod/home-assistant-5db584bd48-4mvwf                          0/1     Running            0                7m56s
homepage                   pod/homepage-58c79b7856-9h59f                                1/1     Running            0                8m8s
...

NAMESPACE                  NAME                                                    READY   UP-TO-DATE   AVAILABLE   AGE
audiobookshelf             deployment.apps/audiobookshelf                          1/1     1            1           258d
cert-manager               deployment.apps/cert-manager                            1/1     1            1           260d
cert-manager               deployment.apps/cert-manager-cainjector                 1/1     1            1           260d
cert-manager               deployment.apps/cert-manager-webhook                    1/1     1            1           260d
code-server                deployment.apps/code-server                             1/1     1            1           255d
default                    deployment.apps/ddns-updater                            1/1     1            1           108d
firefly-iii                deployment.apps/firefly-iii                             1/1     1            1           255d
firefly-iii                deployment.apps/firefly-iii-mysql                       0/1     1            0           255d
home-assistant             deployment.apps/home-assistant                          0/1     1            0           259d
homepage                   deployment.apps/homepage                                1/1     1            1           10d

NAMESPACE                  NAME                                                               DESIRED   CURRENT   READY   AGE
audiobookshelf             replicaset.apps/audiobookshelf-5798dfc5d5                          1         1         1       18h
cert-manager               replicaset.apps/cert-manager-7d67448f59                            1         1         1       260d
cert-manager               replicaset.apps/cert-manager-cainjector-666b8b6b66                 1         1         1       260d
cert-manager               replicaset.apps/cert-manager-webhook-78cb4cf989                    1         1         1       260d
code-server                replicaset.apps/code-server-85785b5848                             1         1         1       7d
default                    replicaset.apps/ddns-updater-75d5df4647                            1         1         1       7d
firefly-iii                replicaset.apps/firefly-iii-f865768bf                              1         1         1       7d
firefly-iii                replicaset.apps/firefly-iii-mysql-659f959f57                       1         1         0       7d
home-assistant             replicaset.apps/home-assistant-5db584bd48                          1         1         0       7d
homepage                   replicaset.apps/homepage-58c79b7856                                1         1         1       9d
```

At this point it seems clear something is not going well with the pods running image
`firefly-iii-mysql`:

``` console hl_lines="4 12"
$ kubectl get all -n firefly-iii 
NAME                                     READY   STATUS             RESTARTS   AGE
pod/firefly-iii-f865768bf-kgjq7          1/1     Running            0          11m
pod/firefly-iii-mysql-659f959f57-ph92z   0/1     ImagePullBackOff   0          10m

NAME                            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/firefly-iii-mysql-svc   NodePort   10.105.106.204   <none>        3306:30306/TCP   255d
service/firefly-iii-svc         NodePort   10.98.129.183    <none>        8080:30080/TCP   255d

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/firefly-iii         1/1     1            1           255d
deployment.apps/firefly-iii-mysql   0/1     1            0           255d

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/firefly-iii-f865768bf          1         1         1       7d
replicaset.apps/firefly-iii-mysql-659f959f57   1         1         0       7d
```

The `ImagePullBackOff` status typically means the required image is not available and
there is no reason to believe that this will resolve it self with tiem. Inspect the
pod's logs:

``` console hl_lines="11"
$ kubectl describe pod firefly-iii-mysql-659f959f57-ph92z -n firefly-iii 
Name:             firefly-iii-mysql-659f959f57-ph92z
Namespace:        firefly-iii
...

Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  ...      ...               ...                  ...                ...
  Normal   Pulling           86s (x5 over 4m29s)  kubelet            Pulling image "yobasystems/alpine-mariadb:latest"
  Warning  Failed            85s (x5 over 4m23s)  kubelet            Failed to pull image "yobasystems/alpine-mariadb:latest": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/yobasystems/alpine-mariadb:latest": failed to resolve image: docker.io/yobasystems/alpine-mariadb:latest: not found
  Warning  Failed            85s (x5 over 4m23s)  kubelet            Error: ErrImagePull
  Normal   BackOff           7s (x15 over 4m22s)  kubelet            Back-off pulling image "yobasystems/alpine-mariadb:latest"
  Warning  Failed            7s (x15 over 4m22s)  kubelet            Error: ImagePullBackOff
```

Kubernetes is reporting that image *tag* "yobasystems/alpine-mariadb:latest" is not
available and indeed it is not!

A thorough search in <https://hub.docker.com/r/yobasystems/alpine-mariadb/tags> shows
there is no such tag as `latest` but there are tags that only contain the CPU
architecture they are built for. The obious chocice is to switch to Linux as much as
you can (e.g. `amd64`), so the manifest in `firefly-iii.yaml` must be updated to pull
that particular image tag:

``` console
$ git diff firefly-iii.yaml 
diff --git a/firefly-iii.yaml b/firefly-iii.yaml
index e6a6f0f..9f9d608 100644
--- a/firefly-iii.yaml
+++ b/firefly-iii.yaml
@@ -64,7 +64,7 @@ spec:
         tier: mysql
     spec:
       containers:
-      - image: yobasystems/alpine-mariadb:latest
+      - image: yobasystems/alpine-mariadb:amd64
         imagePullPolicy: Always
         name: mysql
         env:
```

**Before** applying the update, make a backup copy of the storage usef ror the MySQL
databse:

``` console
# cp -a /home/k8s/firefly-iii/mysql /home/k8s/firefly-iii/mysql-2026-01-11-latest
# du -s /home/k8s/firefly-iii/mysql /home/k8s/firefly-iii/mysql-2026-01-11-latest
220972  /home/k8s/firefly-iii/mysql
220972  /home/k8s/firefly-iii/mysql-2026-01-11-latest
```

Applying the updated manifest gets the pod to correctly find the requested image and
starts the container, so the service is back up again:

``` console hl_lines="5 15"
$ kubectl apply -f firefly-iii.yaml 
namespace/firefly-iii unchanged
persistentvolume/firefly-iii-pv-mysql unchanged
persistentvolumeclaim/firefly-iii-pvc-mysql unchanged
deployment.apps/firefly-iii-mysql configured
service/firefly-iii-mysql-svc unchanged
persistentvolume/firefly-iii-pv-upload unchanged
persistentvolumeclaim/firefly-iii-pvc-upload unchanged
service/firefly-iii-svc unchanged
deployment.apps/firefly-iii unchanged

$ kubectl get all -n firefly-iii
NAME                                    READY   STATUS              RESTARTS   AGE
pod/firefly-iii-f865768bf-kgjq7         1/1     Running             0          15m
pod/firefly-iii-mysql-b8549c77c-4t8mk   0/1     ContainerCreating   0          6s

NAME                            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/firefly-iii-mysql-svc   NodePort   10.105.106.204   <none>        3306:30306/TCP   255d
service/firefly-iii-svc         NodePort   10.98.129.183    <none>        8080:30080/TCP   255d

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/firefly-iii         1/1     1            1           255d
deployment.apps/firefly-iii-mysql   0/1     1            0           255d

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/firefly-iii-f865768bf          1         1         1       7d
replicaset.apps/firefly-iii-mysql-659f959f57   0         0         0       7d
replicaset.apps/firefly-iii-mysql-b8549c77c    1         1         0       6s
```

Inspecting the pod's events shows the image has been successfully pulled:

``` console hl_lines="10-12"
$ kubectl describe pod firefly-iii-mysql-b8549c77c-4t8mk -n firefly-iii 
Name:             firefly-iii-mysql-b8549c77c-4t8mk
Namespace:        firefly-iii
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  19s   default-scheduler  Successfully assigned firefly-iii/firefly-iii-mysql-b8549c77c-4t8mk to octavo
  Normal  Pulling    19s   kubelet            Pulling image "yobasystems/alpine-mariadb:amd64"
  Normal  Pulled     14s   kubelet            Successfully pulled image "yobasystems/alpine-mariadb:amd64" in 4.924s (4.924s including waiting). Image size: 85805520 bytes.
  Normal  Created    14s   kubelet            Created container: mysql
  Normal  Started    14s   kubelet            Started container mysql
```

All this was necessary only because the `latest` tag was *temporarily*
[yobasystems/alpine-mariadb tags](https://hub.docker.com/r/yobasystems/alpine-mariadb/tags),
once the `latest` tag reinstantiated the original manifest would work again.