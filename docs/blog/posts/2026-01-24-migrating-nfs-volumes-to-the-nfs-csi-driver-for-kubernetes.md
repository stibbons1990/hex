---
date: 2026-01-24
categories:
 - linux
 - ubuntu
 - server
 - kubernetes
 - docker
title: Migrating NFS volumes to the NFS CSI driver for Kubernetes
---

NFS volumes have been mounted *the lazy way* as `hostPath` volumes, with the entire NFS
volume being mounted by the host OS. While this works *well enough* in a single-node
cluster, it wouldn't work well in a multi-node cluster and is just not the proper way to
mount NFs volumes in Kubernetes.

For a better, safer and more efficient setup, NFS volumes will now be mounted using the
[NFS CSI driver for Kubernetes](https://github.com/kubernetes-csi/csi-driver-nfs?tab=readme-ov-file#nfs-csi-driver-for-kubernetes).

<!-- more -->

## Enable NFSv4.1 support

It is highly recommended that NFS servers support NFSv4.1 for optimal performance and
security. In a Synology NAS, this setting is in the **Control Panel** under
**File Services > NFS**.

## Install the NFS CSI driver

[Install the NFS CSI driver with Helm](https://github.com/kubernetes-csi/csi-driver-nfs/tree/master/charts#install-csi-driver-with-helm-3)
with this `nfs-csi-values.yaml` for control plane scheduling and modern NFS protocol
support:

!!! k8s "`nfs-csi-values.yaml`"

    ``` yaml
    controller:
      replicas: 1
      runOnControlPlane: true
      dnsPolicy: ClusterFirstWithHostNet

    node:
      dnsPolicy: ClusterFirstWithHostNet

    externalSnapshotter:
      enabled: true
    ```

The use of `dnsPolicy: ClusterFirstWithHostNet` is required when the NFS server is
specified as a hostname, and `externalSnapshotter` is recommended for backups (later).

??? warning "Do not specify `replicas: 2` in a single-node cluster."

    Attempting to run with `replicas: 2` in a single-node customer fails because the
    second pod stays in a `CrashLoopBackOff` loop:

    ``` console
    $ kubectl --namespace=kube-system get pods --selector="app.kubernetes.io/instance=csi-driver-nfs" --watch
    NAME                                READY   STATUS     RESTARTS     AGE
    csi-nfs-controller-5d6c68d9-8lxvq   5/5     Running    0            19s
    csi-nfs-controller-5d6c68d9-bnnhl   4/5     NotReady   1 (3s ago)   18s
    csi-nfs-node-kq85t                  3/3     Running    0            19s
    csi-nfs-controller-5d6c68d9-bnnhl   4/5     CrashLoopBackOff   1 (2s ago)   18s
    csi-nfs-controller-5d6c68d9-bnnhl   4/5     NotReady           2 (15s ago)   31s
    csi-nfs-controller-5d6c68d9-bnnhl   4/5     CrashLoopBackOff   2 (15s ago)   46s
    csi-nfs-controller-5d6c68d9-bnnhl   4/5     NotReady           3 (30s ago)   61s
    csi-nfs-controller-5d6c68d9-bnnhl   4/5     CrashLoopBackOff   3 (15s ago)   75s
    csi-nfs-controller-5d6c68d9-bnnhl   4/5     NotReady           4 (50s ago)   110s
    csi-nfs-controller-5d6c68d9-bnnhl   4/5     CrashLoopBackOff   4 (15s ago)   2m5s

    $ kubectl -n kube-system describe pod csi-nfs-controller-5d6c68d9-bnnhl
    ...
      Warning  BackOff    9s (x15 over 3m5s)    kubelet            Back-off restarting failed container liveness-probe in pod csi-nfs-controller-5d6c68d9-bnnhl_kube-system(30fea10c-5b29-4873-91d4-eed1ae8e3b45)
    ```

    In a single-node cluster, setting `controller.replicas: 2` results in a
    `CrashLoopBackOff` because both controller pods attempt to bind to the same host
    port for health checks or metrics while running on the same physical host. Also, the
    NFS CSI controller uses leader election. While the second pod waits to become the
    leader, it may fail its liveness probe if the probe is incorrectly configured to
    check for an active "leader" state rather than just "running" status.

    The best practice for a single-node cluster that may later be upgraded to a high
    availability multi-node cluster is to start with `replicas: 1` and configure
    **pod anti-affinity** so that, when a new node is added later, when scaling the
    deployment up to `replicas: 2` Kubernetes will automatically place them on
    different nodes:

    !!! k8s "`nfs-csi-values.yaml`"

        ``` yaml hl_lines="6-18"
        controller:
          replicas: 1
          runOnControlPlane: true
          dnsPolicy: ClusterFirstWithHostNet

          # Prepare for future expansion with anti-affinity
          affinity:
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - csi-nfs-controller
                  topologyKey: "kubernetes.io/hostname"

        node:
          dnsPolicy: ClusterFirstWithHostNet

        externalSnapshotter:
          enabled: true
        ```

    Once a new node is added to the cluster, update `nfs-csi-values.yaml` to set
    `replicas: 2` and upgrade the deployment:

    ``` console
    $ helm upgrade csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
        -n kube-system -f nfs-csi-values.yaml
    ```

Install the latest version of the NFS CSI driver in the `kube-system` namespace using the
above `nfs-csi-values.yaml` and check that its opds are all running:

``` console
$ helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts

$ helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
    --namespace kube-system \
    --version 4.12.1 \
    -f nfs-csi-values.yaml
NAME: csi-driver-nfs
LAST DEPLOYED: Sat Jan 24 10:35:53 2026
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The CSI NFS Driver is getting deployed to your cluster.

To check CSI NFS Driver pods status, please run:

  kubectl --namespace=kube-system get pods --selector="app.kubernetes.io/instance=csi-driver-nfs" --watch

$ kubectl --namespace=kube-system get pods --selector="app.kubernetes.io/instance=csi-driver-nfs" --watch
NAME                                READY   STATUS    RESTARTS   AGE
csi-nfs-controller-5d6c68d9-8lxvq   5/5     Running   0          18s
csi-nfs-node-kq85t                  3/3     Running   0          18s
```

!!! warning

    Before continuing to the next saction, ake sure to install the `nfs-common` package
     in the host OS. Otherwise, pods scheduled to nodes lacking `nfs-common` will fail
     to start, typically showing `mount.nfs: command not found` or similar errors.

## Migrate `hostPath` to NFS CSI

The [Navidrome deployment](./2024-10-26-self-hosted-music-streaming-with-navidrome.md)
is used here to illustrate the migration.

To replace each `hostPath` configuration with an NFS CSI mount that targets a specific
subdirectory, start by replacing the `hostPath` element with a `csi` defining the NFS
server and base path and the `subDir` property in  in the `PersistentVolume`:

!!! k8s "`navidrome.yaml` (`PersistentVolume` `navidrome-pv-music`)"
    
    ``` yaml linenums="21" hl_lines="11 13-22" 
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: navidrome-pv-music
      namespace: navidrome
    spec:
      storageClassName: manual
      capacity:
        storage: 100Gi
      accessModes:
        - ReadWriteMany
      persistentVolumeReclaimPolicy: Retain
      csi:
        driver: nfs.csi.k8s.io
        volumeHandle: luggage-music-nfs-octavo
        volumeAttributes:
          server: luggage
          share: /volume1/NetBackup
          subDir: public/audio/Music
      mountOptions:
        - nfsvers=4.1
        - hard
    ```

Changing `accessModes` to `ReadWriteMany` allows multiple nodes to access the volume
simultaneously and adding `subDir: public/audio/Music` make the CSI driver handles the
subdirectory mount so that pods are guaranteed to have no access to other parts of the
NFS volume. CSI drivers often require explicit `mountOptions` like `nfsvers=4.1` or
`hard` to be defined in the `PersistentVolume` `spec` to ensure consistent behavior
across nodes.

The `PersistentVolumeClaim` needs only to change `accessModes` to `ReadWriteMany`:

!!! k8s "`navidrome.yaml` (`PersistentVolumeClaim` `navidrome-pvc-music`)"
    
    ``` yaml linenums="59" hl_lines="10" 
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: navidrome-pvc-music
      namespace: navidrome
    spec:
      storageClassName: manual
      volumeName: navidrome-pv-music
      accessModes:
        - ReadWriteMany
      volumeMode: Filesystem
      resources:
        requests:
          storage: 1Gi
    ```

No changes are required to the pods or deployment, all the differences are abstracted
inside the `PersistentVolume`. What **is required** at this point is to **delete** the
existing `PersistentVolume`, so that a new one may be created with the same name. This
is required to change the storage backend without losing data or creating naming
conflicts; with this "delete and recreate" sequence:

1.  Pre-requisite: set `persistentVolumeReclaimPolicy: Retain` if not already set, so
    that deleting the `PersistentVolume` resource does not trigger the deletion of the
    actual data. This was already set in the original
    [Navidrome deployment](./2024-10-26-self-hosted-music-streaming-with-navidrome.md).

1.  Scale deployments down to `replicas: 0` too stop all the pods using the volume, to
    release the mount:
    ``` console
    $ kubectl scale -n navidrome deployment navidrome --replicas=0
    deployment.apps/navidrome scaled
    ```

1.  Delete the `PersistentVolumeClaim` first, then the `PersistentVolume`:
    ``` console
    $ kubectl -n navidrome delete persistentvolumeclaim navidrome-pvc-music 
    persistentvolumeclaim "navidrome-pvc-music" deleted from navidrome namespace

    $ kubectl delete persistentvolume navidrome-pv-music
    persistentvolume "navidrome-pv-music" deleted
    ```

1.  Recreate the `PersistentVolumeClaim` and `PersistentVolume` by reapplying the
    updated manifest in `navidrome.yaml`. Because the old resources are gone, Kubernetes
    will accept the new configuration as a fresh creation.
    ``` console
    $ kubectl apply -f navidrome.yaml 
    namespace/navidrome unchanged
    persistentvolume/navidrome-pv-data unchanged
    persistentvolume/navidrome-pv-music created
    persistentvolumeclaim/navidrome-pvc-data unchanged
    persistentvolumeclaim/navidrome-pvc-music created
    deployment.apps/navidrome configured
    service/navidrome-svc unchanged
    ```

After a couple of minutes the deployment should be back up, because applying the manifest
scales it back up to `replicas: 1`

``` console
$ kubectl -n navidrome get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/navidrome-64fb55cd88-m6hll   1/1     Running   0          2m17s

NAME                    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/navidrome-svc   NodePort   10.110.51.110   <none>        4533:30533/TCP   270d

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/navidrome   1/1     1            1           270d

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/navidrome-64fb55cd88   1         1         1       3d14h
```

Accessing Navidrome on its web UI and playing music confirms the application still works.

### Repeat for all deployments

Find other deployments using `hostPath` mounts on the host-mounted NFS directory and 
repeat the above steps to migrate them:

``` console
$ grep -rn /home/nas . | cut -f1 -d: | sort -u
./audiobookshelf.yaml
./jellyfin.yaml
./komga.yaml
```

This reveals only a few applications are using the NFS volume, namely
[Audiobookshelf](./2024-02-28-audiobookshelf-on-kubernetes.md),
[Jellyfin](./2025-04-29-jellyfin-on-kubernetes-with-intel-gpu.md) and
[Komga](./2024-05-26-self-hosted-ebook-library-with-komga.md).