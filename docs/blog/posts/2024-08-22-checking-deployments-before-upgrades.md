---
date: 2024-08-22
categories:
 - linux
 - ubuntu
 - server
 - kubernetes
 - docker
title: Checking deployments before upgrading kubeadm clusters
---

I've been meaning to upgrade Kubernetes ever since 
[*Kubernetes Certificate Expired*](2024-03-22-kubectl-certificate-expired.md)
about a year after setting up the
[Kubernetes cluster in lexicon](2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md).

<!-- more -->

[Upgrading kubeadm clusters](https://v1-27.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
is a bit of an involved process and a little intimidating at first,
so it pays to use the tools available to anticipate problems with API
deprecations from one Kubernetes version to the next.

## Install Go

Several of the following tools require Go and in some cases a fairly
recent version, e.g. [`kubeconform`](#kubeconform) requires v1.22.
Ubuntu Server 22.04 packages only provide v1.18, to install newer
versions use `snap` instead:

``` console
# snap install --classic --channel=latest/stable go
go 1.23.0 from Canonical✓ installed
```

For convenience, add `~/go/bin/` to your `PATH`.

## Kubepug

[Deprecations AKA KubePug - Pre UpGrade (Checker)](https://github.com/kubepug/kubepug?tab=readme-ov-file#deprecations--aka-kubepug---pre-upgrade-checker)
is one of the simplest tools to use.

``` console
$ go install github.com/kubepug/kubepug@latest

$ kubepug 2>/dev/null \
  --k8s-version=v1.27 \
  --input-file ~/src/lexicon-deployments/

No deprecated or deleted APIs found

Kubepug validates the APIs using Kubernetes markers. To know what are the deprecated and deleted APIS it checks, please go to https://kubepug.xyz/status/

$ kubepug 2>/dev/null \
  --k8s-version=v1.27 \
  --input-file ~/src/kubernetes-deployments/

No deprecated or deleted APIs found

Kubepug validates the APIs using Kubernetes markers. To know what are the deprecated and deleted APIS it checks, please go to https://kubepug.xyz/status/
```

Same with `--k8s-version=v1.32`.

## Kubent

[kube-no-trouble](https://github.com/doitintl/kube-no-trouble)

``` console
# sh -c "$(curl -sSL https://git.io/install-kubent)"
>>> kubent installation script <<<
> Detecting latest version
> Downloading version 0.7.3
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 13.2M  100 13.2M    0     0  10.6M      0  0:00:01  0:00:01 --:--:-- 44.7M
> Done. kubent was installed to /usr/local/bin/.
```

To check the running environment:

``` console
$ kubent -t 1.27 
12:29AM INF >>> Kube No Trouble `kubent` <<<
12:29AM INF version 0.7.3 (git sha 57480c07b3f91238f12a35d0ec88d9368aae99aa)
12:29AM INF Initializing collectors and retrieving data
12:29AM INF Target K8s version is 1.27.0
12:29AM INF Retrieved 41 resources from collector name=Cluster
12:29AM INF Retrieved 0 resources from collector name="Helm v3"
12:29AM INF Loaded ruleset name=custom.rego.tmpl
12:29AM INF Loaded ruleset name=deprecated-1-16.rego
12:29AM INF Loaded ruleset name=deprecated-1-22.rego
12:29AM INF Loaded ruleset name=deprecated-1-25.rego
12:29AM INF Loaded ruleset name=deprecated-1-26.rego
12:29AM INF Loaded ruleset name=deprecated-1-27.rego
12:29AM INF Loaded ruleset name=deprecated-1-29.rego
12:29AM INF Loaded ruleset name=deprecated-1-32.rego
12:29AM INF Loaded ruleset name=deprecated-future.rego
```

To check manifest files:

``` console
$ kubent -t 1.27 $(find -name "*.yaml")
12:34AM INF >>> Kube No Trouble `kubent` <<<
12:34AM INF version 0.7.3 (git sha 57480c07b3f91238f12a35d0ec88d9368aae99aa)
12:34AM INF Initializing collectors and retrieving data
12:34AM INF Target K8s version is 1.27.0
12:34AM INF Retrieved 41 resources from collector name=Cluster
12:34AM INF Retrieved 0 resources from collector name="Helm v3"
12:34AM INF Loaded ruleset name=custom.rego.tmpl
12:34AM INF Loaded ruleset name=deprecated-1-16.rego
12:34AM INF Loaded ruleset name=deprecated-1-22.rego
12:34AM INF Loaded ruleset name=deprecated-1-25.rego
12:34AM INF Loaded ruleset name=deprecated-1-26.rego
12:34AM INF Loaded ruleset name=deprecated-1-27.rego
12:34AM INF Loaded ruleset name=deprecated-1-29.rego
12:34AM INF Loaded ruleset name=deprecated-1-32.rego
12:34AM INF Loaded ruleset name=deprecated-future.rego
```

Same in `kubernetes-deployments` and `lexicon-deployments`,
with `-t 1.27` and `-t 1.32`.

## kubeconform

[kubeconform](https://github.com/yannh/kubeconform) seems to show the
opposite picture: errors are found even for the currently running
Kubernetes version, which is running all these deployments just fine.

``` console
$ go install github.com/yannh/kubeconform/cmd/kubeconform@latest

$ kubectl  version --output=yaml
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

$ cd ~/src/lexicon-deployments

$ kubeconform -summary -strict -kubernetes-version 1.26.15 $(find -name "*.yaml")
./metallb/ipaddress_pools.yaml - L2Advertisement l2-advert failed validation: could not find schema for L2Advertisement
./metallb/ipaddress_pools.yaml - IPAddressPool production failed validation: could not find schema for IPAddressPool
./metallb/metallb-native.yaml - CustomResourceDefinition addresspools.metallb.io failed validation: could not find schema for CustomResourceDefinition
./metallb/metallb-native.yaml - CustomResourceDefinition bgppeers.metallb.io failed validation: could not find schema for CustomResourceDefinition
./metallb/metallb-native.yaml - CustomResourceDefinition communities.metallb.io failed validation: could not find schema for CustomResourceDefinition
./metallb/metallb-native.yaml - CustomResourceDefinition ipaddresspools.metallb.io failed validation: could not find schema for CustomResourceDefinition
./metallb/metallb-native.yaml - CustomResourceDefinition l2advertisements.metallb.io failed validation: could not find schema for CustomResourceDefinition
./metallb/metallb-native.yaml - CustomResourceDefinition bfdprofiles.metallb.io failed validation: could not find schema for CustomResourceDefinition
./metallb/metallb-native.yaml - CustomResourceDefinition bgpadvertisements.metallb.io failed validation: could not find schema for CustomResourceDefinition
./gitea/values-with-ingress.yaml - failed validation: error while parsing: missing 'kind' key
./gitea/values.yaml - failed validation: error while parsing: missing 'kind' key
Summary: 71 resources found in 9 files - Valid: 60, Invalid: 0, Errors: 11, Skipped: 0

$ cd ~/src/kubernetes-deployments

$ kubeconform \
  -summary -strict \
  -kubernetes-version 1.26.15 \
  $(find -name "*.yaml")
./1.26/metallb/metallb-native.yaml - CustomResourceDefinition bfdprofiles.metallb.io failed validation: could not find schema for CustomResourceDefinition
./1.26/metallb/metallb-native.yaml - CustomResourceDefinition bgpadvertisements.metallb.io failed validation: could not find schema for CustomResourceDefinition
./1.26/metallb/metallb-native.yaml - CustomResourceDefinition bgppeers.metallb.io failed validation: could not find schema for CustomResourceDefinition
./1.26/metallb/metallb-native.yaml - CustomResourceDefinition communities.metallb.io failed validation: could not find schema for CustomResourceDefinition
./1.26/metallb/metallb-native.yaml - CustomResourceDefinition ipaddresspools.metallb.io failed validation: could not find schema for CustomResourceDefinition
./1.26/metallb/metallb-native.yaml - CustomResourceDefinition servicel2statuses.metallb.io failed validation: could not find schema for CustomResourceDefinition
./1.26/metallb/metallb-native.yaml - CustomResourceDefinition l2advertisements.metallb.io failed validation: could not find schema for CustomResourceDefinition
./1.26/metallb/ipaddress-pool-lexicon.yaml - IPAddressPool production failed validation: could not find schema for IPAddressPool
./1.26/metallb/ipaddress-pool-rapture.yaml - IPAddressPool rapture-pool failed validation: could not find schema for IPAddressPool
./1.26/metallb/ipaddress-pool-lexicon.yaml - L2Advertisement l2-advert failed validation: could not find schema for L2Advertisement
./1.26/metallb/ipaddress-pool-rapture.yaml - L2Advertisement l2-advert failed validation: could not find schema for L2Advertisement
Summary: 152 resources found in 16 files - Valid: 141, Invalid: 0, Errors: 11, Skipped: 0
```

Even if the above errors can be safely ignored, many more are returned
when running with a higher target version, e.g. 1.32:

``` console
$ cd ~/src/lexicon-deployments

$ kubeconform -summary -strict -kubernetes-version 1.32.2 */*.yaml
dashboard/kubernetes-dashboard.yaml - Service kubernetes-dashboard failed validation: could not find schema for Service
dashboard/kubernetes-dashboard-ingress.yaml - Ingress kubernetes-dashboard-ingress failed validation: could not find schema for Ingress
dashboard/kubernetes-dashboard.yaml - Namespace kubernetes-dashboard failed validation: could not find schema for Namespace
dashboard/kubernetes-dashboard.yaml - ServiceAccount kubernetes-dashboard failed validation: could not find schema for ServiceAccount
dashboard/kubernetes-dashboard.yaml - Secret kubernetes-dashboard-certs failed validation: could not find schema for Secret
dashboard/kubernetes-dashboard.yaml - Secret kubernetes-dashboard-key-holder failed validation: could not find schema for Secret
dashboard/kubernetes-dashboard.yaml - Secret kubernetes-dashboard-csrf failed validation: could not find schema for Secret
dashboard/kubernetes-dashboard.yaml - ConfigMap kubernetes-dashboard-settings failed validation: could not find schema for ConfigMap
dashboard/kubernetes-dashboard.yaml - Role kubernetes-dashboard failed validation: could not find schema for Role
dashboard/kubernetes-dashboard.yaml - RoleBinding kubernetes-dashboard failed validation: could not find schema for RoleBinding
dashboard/kubernetes-dashboard.yaml - Service dashboard-metrics-scraper failed validation: could not find schema for Service
dashboard/kubernetes-dashboard.yaml - ClusterRole kubernetes-dashboard failed validation: could not find schema for ClusterRole
gitea/values-with-ingress.yaml - failed validation: error while parsing: missing 'kind' key
gitea/values.yaml - failed validation: error while parsing: missing 'kind' key
ingress-nginx/nginx-ingress-controller-deploy.yaml - Namespace ingress-nginx failed validation: could not find schema for Namespace
ingress-nginx/nginx-ingress-controller-deploy.yaml - ServiceAccount ingress-nginx failed validation: could not find schema for ServiceAccount
ingress-nginx/nginx-ingress-controller-deploy.yaml - ServiceAccount ingress-nginx-admission failed validation: could not find schema for ServiceAccount
ingress-nginx/nginx-ingress-controller-deploy.yaml - Role ingress-nginx failed validation: could not find schema for Role
ingress-nginx/nginx-ingress-controller-deploy.yaml - Role ingress-nginx-admission failed validation: could not find schema for Role
ingress-nginx/nginx-ingress-controller-deploy.yaml - ClusterRole ingress-nginx failed validation: could not find schema for ClusterRole
ingress-nginx/nginx-ingress-controller-deploy.yaml - ClusterRole ingress-nginx-admission failed validation: could not find schema for ClusterRole
ingress-nginx/nginx-ingress-controller-deploy.yaml - RoleBinding ingress-nginx failed validation: could not find schema for RoleBinding
ingress-nginx/nginx-ingress-controller-deploy.yaml - RoleBinding ingress-nginx-admission failed validation: could not find schema for RoleBinding
dashboard/kubernetes-dashboard.yaml - ClusterRoleBinding kubernetes-dashboard failed validation: could not find schema for ClusterRoleBinding
ingress-nginx/nginx-ingress-controller-deploy.yaml - ClusterRoleBinding ingress-nginx-admission failed validation: could not find schema for ClusterRoleBinding
ingress-nginx/nginx-ingress-controller-deploy.yaml - ConfigMap ingress-nginx-controller failed validation: could not find schema for ConfigMap
ingress-nginx/nginx-ingress-controller-deploy.yaml - Service ingress-nginx-controller failed validation: could not find schema for Service
ingress-nginx/nginx-ingress-controller-deploy.yaml - ClusterRoleBinding ingress-nginx failed validation: could not find schema for ClusterRoleBinding
ingress-nginx/nginx-ingress-controller-deploy.yaml - Service ingress-nginx-controller-admission failed validation: could not find schema for Service
dashboard/kubernetes-dashboard.yaml - Deployment kubernetes-dashboard failed validation: could not find schema for Deployment
ingress-nginx/nginx-ingress-controller-deploy.yaml - Deployment ingress-nginx-controller failed validation: could not find schema for Deployment
dashboard/kubernetes-dashboard.yaml - Deployment dashboard-metrics-scraper failed validation: could not find schema for Deployment
ingress-nginx/nginx-ingress-controller-deploy.yaml - Job ingress-nginx-admission-create failed validation: could not find schema for Job
ingress-nginx/nginx-ingress-controller-deploy.yaml - Job ingress-nginx-admission-patch failed validation: could not find schema for Job
ingress-nginx/nginx-ingress-controller-deploy.yaml - ValidatingWebhookConfiguration ingress-nginx-admission failed validation: could not find schema for ValidatingWebhookConfiguration
metallb/metallb-native.yaml - Namespace metallb-system failed validation: could not find schema for Namespace
ingress-nginx/nginx-ingress-controller-deploy.yaml - IngressClass nginx failed validation: could not find schema for IngressClass
metallb/ipaddress_pools.yaml - L2Advertisement l2-advert failed validation: could not find schema for L2Advertisement
metallb/ipaddress_pools.yaml - IPAddressPool production failed validation: could not find schema for IPAddressPool
metallb/metallb-native.yaml - CustomResourceDefinition addresspools.metallb.io failed validation: could not find schema for CustomResourceDefinition
metallb/metallb-native.yaml - CustomResourceDefinition communities.metallb.io failed validation: could not find schema for CustomResourceDefinition
metallb/metallb-native.yaml - CustomResourceDefinition bgpadvertisements.metallb.io failed validation: could not find schema for CustomResourceDefinition
metallb/metallb-native.yaml - CustomResourceDefinition bgppeers.metallb.io failed validation: could not find schema for CustomResourceDefinition
metallb/metallb-native.yaml - CustomResourceDefinition bfdprofiles.metallb.io failed validation: could not find schema for CustomResourceDefinition
metallb/metallb-native.yaml - ServiceAccount controller failed validation: could not find schema for ServiceAccount
metallb/metallb-native.yaml - ServiceAccount speaker failed validation: could not find schema for ServiceAccount
metallb/metallb-native.yaml - CustomResourceDefinition ipaddresspools.metallb.io failed validation: could not find schema for CustomResourceDefinition
metallb/metallb-native.yaml - Role controller failed validation: could not find schema for Role
metallb/metallb-native.yaml - Role pod-lister failed validation: could not find schema for Role
metallb/metallb-native.yaml - CustomResourceDefinition l2advertisements.metallb.io failed validation: could not find schema for CustomResourceDefinition
metallb/metallb-native.yaml - ClusterRole metallb-system:controller failed validation: could not find schema for ClusterRole
metallb/metallb-native.yaml - RoleBinding controller failed validation: could not find schema for RoleBinding
metallb/metallb-native.yaml - ClusterRole metallb-system:speaker failed validation: could not find schema for ClusterRole
metallb/metallb-native.yaml - RoleBinding pod-lister failed validation: could not find schema for RoleBinding
metallb/metallb-native.yaml - ClusterRoleBinding metallb-system:controller failed validation: could not find schema for ClusterRoleBinding
metallb/metallb-native.yaml - ClusterRoleBinding metallb-system:speaker failed validation: could not find schema for ClusterRoleBinding
metallb/metallb-native.yaml - Secret webhook-server-cert failed validation: could not find schema for Secret
metallb/metallb-native.yaml - Service webhook-service failed validation: could not find schema for Service
metallb/metallb-native.yaml - Deployment controller failed validation: could not find schema for Deployment
metallb/web-app-demo.yaml - Namespace web failed validation: could not find schema for Namespace
metallb/web-app-demo.yaml - Service web-server-service failed validation: could not find schema for Service
metallb/web-app-demo.yaml - Deployment web-server failed validation: could not find schema for Deployment
metallb/metallb-native.yaml - ValidatingWebhookConfiguration metallb-webhook-configuration failed validation: could not find schema for ValidatingWebhookConfiguration
metallb/metallb-native.yaml - DaemonSet speaker failed validation: could not find schema for DaemonSet
Summary: 64 resources found in 8 files - Valid: 0, Invalid: 0, Errors: 64, Skipped: 0

$ cd ~/src/kubernetes-deployments

$ kubeconform -summary -strict -kubernetes-version 1.32.2 */*/*.yaml
1.26/dashboard/admin-sa-rbac.yaml - ClusterRoleBinding k8sadmin failed validation: could not find schema for ClusterRoleBinding
1.26/dashboard/admin-sa-rbac.yaml - ServiceAccount k8sadmin failed validation: could not find schema for ServiceAccount
1.26/dashboard/admin-sa-rbac.yaml - Secret k8sadmin-token failed validation: could not find schema for Secret
1.26/dashboard/kubernetes-dashboard.yaml - Secret kubernetes-dashboard-certs failed validation: could not find schema for Secret
1.26/dashboard/kubernetes-dashboard.yaml - Secret kubernetes-dashboard-csrf failed validation: could not find schema for Secret
1.26/dashboard/kubernetes-dashboard.yaml - Secret kubernetes-dashboard-key-holder failed validation: could not find schema for Secret
1.26/dashboard/kubernetes-dashboard.yaml - Namespace kubernetes-dashboard failed validation: could not find schema for Namespace
1.26/dashboard/kubernetes-dashboard.yaml - ServiceAccount kubernetes-dashboard failed validation: could not find schema for ServiceAccount
1.26/dashboard/kubernetes-dashboard.yaml - Service kubernetes-dashboard failed validation: could not find schema for Service
1.26/dashboard/kubernetes-dashboard.yaml - Role kubernetes-dashboard failed validation: could not find schema for Role
1.26/dashboard/kubernetes-dashboard.yaml - ClusterRoleBinding kubernetes-dashboard failed validation: could not find schema for ClusterRoleBinding
1.26/dashboard/kubernetes-dashboard.yaml - ClusterRole kubernetes-dashboard failed validation: could not find schema for ClusterRole
1.26/dashboard/kubernetes-dashboard.yaml - Service dashboard-metrics-scraper failed validation: could not find schema for Service
1.26/dashboard/kubernetes-dashboard.yaml - ConfigMap kubernetes-dashboard-settings failed validation: could not find schema for ConfigMap
1.26/lexicon/audiobookshelf.yaml - Namespace audiobookshelf failed validation: could not find schema for Namespace
1.26/dashboard/kubernetes-dashboard.yaml - Deployment kubernetes-dashboard failed validation: could not find schema for Deployment
1.26/dashboard/kubernetes-dashboard.yaml - Deployment dashboard-metrics-scraper failed validation: could not find schema for Deployment
1.26/dashboard/kubernetes-dashboard.yaml - RoleBinding kubernetes-dashboard failed validation: could not find schema for RoleBinding
1.26/lexicon/audiobookshelf.yaml - PersistentVolume audiobookshelf-pv-config failed validation: could not find schema for PersistentVolume
1.26/lexicon/audiobookshelf.yaml - PersistentVolume audiobookshelf-pv-metadata failed validation: could not find schema for PersistentVolume
1.26/lexicon/audiobookshelf.yaml - PersistentVolume audiobookshelf-pv-audiobooks failed validation: could not find schema for PersistentVolume
1.26/lexicon/audiobookshelf.yaml - PersistentVolume audiobookshelf-pv-podcasts failed validation: could not find schema for PersistentVolume
1.26/lexicon/audiobookshelf.yaml - PersistentVolumeClaim audiobookshelf-pvc-config failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/audiobookshelf.yaml - Deployment audiobookshelf failed validation: could not find schema for Deployment
1.26/lexicon/audiobookshelf.yaml - Service audiobookshelf-svc failed validation: could not find schema for Service
1.26/lexicon/audiobookshelf.yaml - PersistentVolumeClaim audiobookshelf-pvc-metadata failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/firefly-iii.yaml - Namespace firefly-iii failed validation: could not find schema for Namespace
1.26/lexicon/audiobookshelf.yaml - PersistentVolumeClaim audiobookshelf-pvc-audiobooks failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/audiobookshelf.yaml - PersistentVolumeClaim audiobookshelf-pvc-podcasts failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/firefly-iii.yaml - PersistentVolume firefly-iii-pv-mysql failed validation: could not find schema for PersistentVolume
1.26/lexicon/firefly-iii.yaml - PersistentVolumeClaim firefly-iii-pvc-mysql failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/firefly-iii.yaml - Deployment firefly-iii-mysql failed validation: could not find schema for Deployment
1.26/lexicon/firefly-iii.yaml - PersistentVolume firefly-iii-pv-upload failed validation: could not find schema for PersistentVolume
1.26/lexicon/firefly-iii.yaml - PersistentVolumeClaim firefly-iii-pvc-upload failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/firefly-iii.yaml - Service firefly-iii-mysql-svc failed validation: could not find schema for Service
1.26/lexicon/firefly-iii.yaml - Service firefly-iii-svc failed validation: could not find schema for Service
1.26/lexicon/homebox.yaml - Namespace homebox failed validation: could not find schema for Namespace
1.26/lexicon/homebox.yaml - PersistentVolume homebox-pv-data failed validation: could not find schema for PersistentVolume
1.26/lexicon/firefly-iii.yaml - Deployment firefly-iii failed validation: could not find schema for Deployment
1.26/lexicon/homebox.yaml - PersistentVolumeClaim homebox-pvc-data failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/homebox.yaml - Service homebox-svc failed validation: could not find schema for Service
1.26/lexicon/homebox.yaml - Deployment homebox failed validation: could not find schema for Deployment
1.26/lexicon/kavita.yaml - Namespace kavita failed validation: could not find schema for Namespace
1.26/lexicon/kavita.yaml - PersistentVolume kavita-pv-config failed validation: could not find schema for PersistentVolume
1.26/lexicon/kavita.yaml - PersistentVolume kavita-pv-books failed validation: could not find schema for PersistentVolume
1.26/lexicon/kavita.yaml - PersistentVolumeClaim kavita-pvc-config failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/kavita.yaml - PersistentVolumeClaim kavita-pvc-books failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/kavita.yaml - Deployment kavita failed validation: could not find schema for Deployment
1.26/lexicon/kavita.yaml - Service kavita-svc failed validation: could not find schema for Service
1.26/lexicon/audiobookshelf.yaml - Ingress audiobookshelf-ingress failed validation: could not find schema for Ingress
1.26/lexicon/komga.yaml - Namespace komga failed validation: could not find schema for Namespace
1.26/lexicon/komga.yaml - PersistentVolume komga-pv-config failed validation: could not find schema for PersistentVolume
1.26/lexicon/komga.yaml - PersistentVolume komga-pv-books failed validation: could not find schema for PersistentVolume
1.26/lexicon/komga.yaml - PersistentVolumeClaim komga-pvc-config failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/kavita.yaml - Ingress kavita-ingress failed validation: could not find schema for Ingress
1.26/lexicon/firefly-iii.yaml - Ingress firefly-iii-ingress failed validation: could not find schema for Ingress
1.26/lexicon/homebox.yaml - Ingress homebox-ingress failed validation: could not find schema for Ingress
1.26/lexicon/komga.yaml - PersistentVolumeClaim komga-pvc-books failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/komga.yaml - Deployment komga failed validation: could not find schema for Deployment
1.26/lexicon/komga.yaml - Service komga-svc failed validation: could not find schema for Service
1.26/lexicon/komga.yaml - Ingress komga-ingress failed validation: could not find schema for Ingress
1.26/lexicon/minecraft-server.yaml - Namespace minecraft-server failed validation: could not find schema for Namespace
1.26/lexicon/minecraft-server.yaml - Service minecraft-server failed validation: could not find schema for Service
1.26/lexicon/minecraft-server.yaml - PersistentVolume minecraft-server-pv failed validation: could not find schema for PersistentVolume
1.26/lexicon/minecraft-server.yaml - PersistentVolumeClaim minecraft-server-pv-claim failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/plex-media-server.yaml - Namespace plexserver failed validation: could not find schema for Namespace
1.26/lexicon/minecraft-server.yaml - Deployment minecraft-server failed validation: could not find schema for Deployment
1.26/lexicon/plex-media-server.yaml - PersistentVolume plexserver-pv-config failed validation: could not find schema for PersistentVolume
1.26/lexicon/plex-media-server.yaml - PersistentVolume plexserver-pv-data-depot failed validation: could not find schema for PersistentVolume
1.26/lexicon/plex-media-server.yaml - PersistentVolume plexserver-pv-data-video failed validation: could not find schema for PersistentVolume
1.26/lexicon/plex-media-server.yaml - PersistentVolumeClaim plexserver-pvc-config failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/plex-media-server.yaml - PersistentVolumeClaim plexserver-pvc-data-depot failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/plex-media-server.yaml - PersistentVolumeClaim plexserver-pvc-data-video failed validation: could not find schema for PersistentVolumeClaim
1.26/lexicon/plex-media-server.yaml - Service plex-udp failed validation: could not find schema for Service
1.26/lexicon/plex-media-server.yaml - Service plex-tcp failed validation: could not find schema for Service
1.26/lexicon/plex-media-server.yaml - Deployment plexserver failed validation: could not find schema for Deployment
1.26/metallb/ipaddress-pool-rapture.yaml - IPAddressPool rapture-pool failed validation: could not find schema for IPAddressPool
1.26/metallb/metallb-native.yaml - Namespace metallb-system failed validation: could not find schema for Namespace
1.26/metallb/ipaddress-pool-lexicon.yaml - IPAddressPool production failed validation: could not find schema for IPAddressPool
1.26/metallb/ipaddress-pool-rapture.yaml - L2Advertisement l2-advert failed validation: could not find schema for L2Advertisement
1.26/metallb/ipaddress-pool-lexicon.yaml - L2Advertisement l2-advert failed validation: could not find schema for L2Advertisement
1.26/metallb/metallb-native.yaml - CustomResourceDefinition bfdprofiles.metallb.io failed validation: could not find schema for CustomResourceDefinition
1.26/metallb/metallb-native.yaml - CustomResourceDefinition ipaddresspools.metallb.io failed validation: could not find schema for CustomResourceDefinition
1.26/metallb/metallb-native.yaml - CustomResourceDefinition bgpadvertisements.metallb.io failed validation: could not find schema for CustomResourceDefinition
1.26/metallb/metallb-native.yaml - CustomResourceDefinition communities.metallb.io failed validation: could not find schema for CustomResourceDefinition
1.26/metallb/metallb-native.yaml - CustomResourceDefinition bgppeers.metallb.io failed validation: could not find schema for CustomResourceDefinition
1.26/metallb/metallb-native.yaml - ServiceAccount controller failed validation: could not find schema for ServiceAccount
1.26/metallb/metallb-native.yaml - CustomResourceDefinition l2advertisements.metallb.io failed validation: could not find schema for CustomResourceDefinition
1.26/metallb/metallb-native.yaml - ServiceAccount speaker failed validation: could not find schema for ServiceAccount
1.26/metallb/metallb-native.yaml - CustomResourceDefinition servicel2statuses.metallb.io failed validation: could not find schema for CustomResourceDefinition
1.26/metallb/metallb-native.yaml - Role controller failed validation: could not find schema for Role
1.26/metallb/metallb-native.yaml - Role pod-lister failed validation: could not find schema for Role
1.26/metallb/metallb-native.yaml - ClusterRole metallb-system:speaker failed validation: could not find schema for ClusterRole
1.26/metallb/metallb-native.yaml - RoleBinding controller failed validation: could not find schema for RoleBinding
1.26/metallb/metallb-native.yaml - ClusterRole metallb-system:controller failed validation: could not find schema for ClusterRole
1.26/metallb/metallb-native.yaml - ClusterRoleBinding metallb-system:controller failed validation: could not find schema for ClusterRoleBinding
1.26/metallb/metallb-native.yaml - RoleBinding pod-lister failed validation: could not find schema for RoleBinding
1.26/metallb/metallb-native.yaml - ConfigMap metallb-excludel2 failed validation: could not find schema for ConfigMap
1.26/metallb/metallb-native.yaml - ClusterRoleBinding metallb-system:speaker failed validation: could not find schema for ClusterRoleBinding
1.26/metallb/metallb-native.yaml - Secret metallb-webhook-cert failed validation: could not find schema for Secret
1.26/metallb/metallb-native.yaml - Service metallb-webhook-service failed validation: could not find schema for Service
1.26/metallb/metallb-native.yaml - Deployment controller failed validation: could not find schema for Deployment
1.26/rapture/audiobookshelf.yaml - Namespace audiobookshelf failed validation: could not find schema for Namespace
1.26/rapture/audiobookshelf.yaml - PersistentVolume audiobookshelf-pv-metadata failed validation: could not find schema for PersistentVolume
1.26/rapture/audiobookshelf.yaml - PersistentVolume audiobookshelf-pv-config failed validation: could not find schema for PersistentVolume
1.26/rapture/audiobookshelf.yaml - PersistentVolume audiobookshelf-pv-audiobooks failed validation: could not find schema for PersistentVolume
1.26/rapture/audiobookshelf.yaml - PersistentVolumeClaim audiobookshelf-pvc-metadata failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/audiobookshelf.yaml - PersistentVolumeClaim audiobookshelf-pvc-config failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/audiobookshelf.yaml - PersistentVolumeClaim audiobookshelf-pvc-audiobooks failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/audiobookshelf.yaml - Service audiobookshelf-svc failed validation: could not find schema for Service
1.26/rapture/photoprism.yaml - Namespace photoprism failed validation: could not find schema for Namespace
1.26/rapture/audiobookshelf.yaml - Deployment audiobookshelf failed validation: could not find schema for Deployment
1.26/rapture/photoprism.yaml - Secret photoprism-secrets failed validation: could not find schema for Secret
1.26/rapture/photoprism.yaml - PersistentVolume photoprism-pv-originals failed validation: could not find schema for PersistentVolume
1.26/rapture/photoprism.yaml - PersistentVolume photoprism-pv-import failed validation: could not find schema for PersistentVolume
1.26/rapture/photoprism.yaml - PersistentVolume photoprism-pv-storage failed validation: could not find schema for PersistentVolume
1.26/rapture/photoprism.yaml - PersistentVolumeClaim photoprism-pvc-originals failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/photoprism.yaml - PersistentVolumeClaim photoprism-pvc-import failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/photoprism.yaml - PersistentVolumeClaim photoprism-pvc-storage failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/photoprism.yaml - Service photoprism failed validation: could not find schema for Service
1.26/rapture/plex-media-server.yaml - Namespace plexserver failed validation: could not find schema for Namespace
1.26/rapture/plex-media-server.yaml - PersistentVolume plexserver-pv-config failed validation: could not find schema for PersistentVolume
1.26/rapture/plex-media-server.yaml - PersistentVolume plexserver-pv-data-audio failed validation: could not find schema for PersistentVolume
1.26/rapture/plex-media-server.yaml - PersistentVolume plexserver-pv-data-video failed validation: could not find schema for PersistentVolume
1.26/rapture/plex-media-server.yaml - PersistentVolumeClaim plexserver-pvc-config failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/plex-media-server.yaml - PersistentVolumeClaim plexserver-pvc-data-audio failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/plex-media-server.yaml - PersistentVolumeClaim plexserver-pvc-data-video failed validation: could not find schema for PersistentVolumeClaim
1.26/rapture/plex-media-server.yaml - Deployment plexserver failed validation: could not find schema for Deployment
1.26/rapture/plex-media-server.yaml - Service plex-udp failed validation: could not find schema for Service
1.26/rapture/plex-media-server.yaml - Service plex-tcp failed validation: could not find schema for Service
1.26/metallb/metallb-native.yaml - DaemonSet speaker failed validation: could not find schema for DaemonSet
1.26/rapture/photoprism.yaml - StatefulSet photoprism failed validation: could not find schema for StatefulSet
1.26/metallb/metallb-native.yaml - ValidatingWebhookConfiguration metallb-webhook-configuration failed validation: could not find schema for ValidatingWebhookConfiguration
Summary: 133 resources found in 15 files - Valid: 0, Invalid: 0, Errors: 133, Skipped: 0
```
