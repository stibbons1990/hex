---
date: 2026-01-25
categories:
 - linux
 - ubuntu
 - server
 - kubernetes
 - docker
title: Migrating Kubernetes volumes to Longhorn
---

[Migrating NFS volumes to the NFS CSI driver](./2026-01-24-migrating-nfs-volumes-to-the-nfs-csi-driver-for-kubernetes.md)
was an easy step forward preparing the single-node Kubernetes cluster to be upgraded to
an **Active-Active High Availability** cluster.
The next step in that direction is to migrate volumes currently implemented
(*the lazy way*) with `hostPath` pointed to local NVMe SSD storage to a distributed file
system, while still leveraging the SSDs speed as high-availability distributed volumes
replicated across every node's local NVMe SSDs.

<!-- more -->

## Motivation

While NFS volumes are the best choice for bulk data (media, backups, etc.) these are too
slow for mission-critical configuration files and databases. Longhorn is better for these
because it can mirror data across all nodes, if one fails the pod instantly restarts on
another node with its data intact, and whenever adding a new node Longhorn automatically
rebalances replicas to utilize the new storage. Moving forward, Longhorn can also use
the Synology NAS as a "Backup Target" to make additional backups.

## Prepare Host OS

Run these commands on all current and future nodes to install Longhorn’s required dependencies

``` console
# apt install -y open-iscsi util-linux dmsetup

# modprobe iscsi_tcp
# modprobe dm_crypt
# echo "iscsi_tcp" | tee -a /etc/modules-load.d/longhorn.conf
iscsi_tcp
# echo "dm_crypt" | tee -a /etc/modules-load.d/longhorn.conf
dm_crypt

# systemctl enable --now iscsid
Synchronizing state of iscsid.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable iscsid
Created symlink /etc/systemd/system/sysinit.target.wants/iscsid.service → /usr/lib/systemd/system/iscsid.service.

# systemctl status iscsid
● iscsid.service - iSCSI initiator daemon (iscsid)
     Loaded: loaded (/usr/lib/systemd/system/iscsid.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-01-24 15:34:28 CET; 19s ago
TriggeredBy: ● iscsid.socket
       Docs: man:iscsid(8)
    Process: 1256258 ExecStartPre=/usr/lib/open-iscsi/startup-checks.sh (code=exited, status=0/SUCCESS)
    Process: 1256279 ExecStart=/usr/sbin/iscsid (code=exited, status=0/SUCCESS)
   Main PID: 1256334 (iscsid)
      Tasks: 2 (limit: 37734)
     Memory: 3.6M (peak: 3.9M)
        CPU: 27ms
     CGroup: /system.slice/iscsid.service
             ├─1256333 /usr/sbin/iscsid
             └─1256334 /usr/sbin/iscsid

Jan 24 15:34:28 octavo systemd[1]: Starting iscsid.service - iSCSI initiator daemon (iscsid)...
Jan 24 15:34:28 octavo iscsid[1256279]: iSCSI logger with pid=1256333 started!
Jan 24 15:34:28 octavo systemd[1]: Started iscsid.service - iSCSI initiator daemon (iscsid).
Jan 24 15:34:29 octavo iscsid[1256333]: iSCSI daemon with pid=1256334 started!
```

??? note "Additional dependencies for optional features."

    *   `cryptsetup` and `dm_crypt` would be required to enable **Volume Encryption**.
        `cryptsetup` uses the `dm_crypt` kernel module to manage LUKS2-formatted
        encrypted volumes (alss used previously to
        [encrypt external SSD](./2024-01-10-encrypting-external-ssd.md)), ensuring that
        data is secured at rest before being written to the physical SSDs
    *   `jq` is not used by the Longhorn storage engine itself during runtime, but it is
        useful to parse JSON output and would be needed for running Longhorn Environment
        Check scripts to verify nodes before installation.
    *   `nfs-common` will be needed to make backups to NFS volumes; already installed for
        [Migrating NFS volumes to the NFS CSI driver](./2026-01-24-migrating-nfs-volumes-to-the-nfs-csi-driver-for-kubernetes.md)

### Migration from Btrfs to XFS

Longhorn
[Installation Requirements](https://longhorn.io/docs/1.10.1/deploy/install/#installation-requirements)
includes a host filesystem that supports the `file extents` feature to store the data,
`ext4` and `XFS` being the **only ones supported**; this presents a challenge because
both local disks in `octavo` are formatted as `Btrfs` and all workloads depend on at
least one of them.

While there are adjustments that can be made to `Btrfs` volumes to minimize issues with
Longhorn, ultimately `Btrfs` being unsupported means a kernel update or a specific IO
pattern could still lead to volumes becoming stuck in Read-Only mode. Alternatives to
Longhorn such as Ceph (via Rook Operator), GlusterFS (via various CSI drivers) or
OpenEBS LocalPV also have enough incompatibilities with `Btrfs` volumes that would not
serve the purpose.

#### Remove use of the SATA SSD

!!! warning

    Stop all `crontab` jobs and other scripts that may be syncing files to or from any
    file sytem in the NVMe or SATA SSDS. Make sure that scripts running periodically
    (e.g. those listed by `crontab -l`) will not run or write data under `/home/ssd`
    since that would write data to the NVMe SSD and may accidentally fill it up.

The only use of the SATA SSD is for media files that are replicated **from** the NAs,
so there is nothing in the SATA SSD that needs to be copied outside of it. It is enough
to migrate the one volume using the SATA SSD to a NFS CSI mount and the disk is ready to
format.

Create a new XFS file system on the SATA SSD and mount it in a temporary directory:

``` console
# umount /home/ssd 
# mkfs.xfs -f /dev/sda
meta-data=/dev/sda               isize=512    agcount=4, agsize=244188662 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=976754646, imaxpct=5
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=476930, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.

# blkid /dev/sda
/dev/sda: UUID="76347c52-c635-49b3-baa3-4ea98e41b4a4" BLOCK_SIZE="4096" TYPE="xfs"

# mkdir -p /mnt/sata_temp
# mount /dev/sda /mnt/sata_temp
```

!!! warning

    Remove the line in `/etc/fstab` to mount the old file system on `/home/ssd` or else
    the system will no longer boot.

#### Copy NVMe to SATA SSD

Since the `/home` partitiion is critical for all workloads, the migration must be
performed with the cluster services stopped to ensure data consistency. To minimize
down-time for the affected services, at least the largest part of the data can be
copied while workloads are running because it is *mostly read-only*:

``` console
# time rsync --bwlimit=500000 -aHAXv /home/depot /mnt/sata_temp/

sent 921,117,494,040 bytes  received 741,595 bytes  364,438,471.07 bytes/sec
total size is 920,889,625,124  speedup is 1.00

real    42m7.041s
user    7m56.286s
sys     22m42.006s
```

!!! warning

    Writting to the [problematic Crucial MX500 SSD](./2022-10-12-crucial-mx500-ssd-found-problematic.md)
    requires hard-limiting the bandwidth to 500 MB/s with `--bwlimit=500000`

This takes 42 minutes to copy over 858 GB of `Podcasts`; that's 42 minutes less of
down-time with all workloads down for the complete migration. Additional time can be
saved if other directories are found to be unnecessary to move; as it happens 240 GB
were found left back under `/home/k8s/photos` with no workload using them and additional
40 GB were found under `/home/k8s/minecraft-server-backups` that were also out of use.

Once most of the data has been transfered, all Kubernetes workloads and services must
be stopped to finalize the transfer. It is actually necessary to first **drain** the
node, to make sure all pods are stopped, otherwise pods will continue running and
potentially writting to the files in the current `/home` partition.

``` console
# kubectl drain --ignore-daemonsets --delete-emptydir-data --force octavo
# systemctl stop kubelet
# systemctl stop containerd
```

Once all the processes are stopped, `lsof` should report that not a single process is
using files under `/home` even though it may not yet be possible to unmount it.

``` console
# time lsof +D /home 

real    0m8.257s
user    0m1.813s
sys     0m7.057s

# umount /home
umount: /home: target is busy.
```

`/home` cannot be unmounted yet because the NFS volume from the Synology NAS is still
mounted as `/home/nas` and before that one can be unmounted it is necessary to stop the
[Continuous Monitoring](../../projects/conmon.md) service:

``` console
# systemctl stop conmon
# umount /home/nas
# umount /home
```

!!! note 

    Add `-x` to the `rsync` command flags to avoid copying files from other file systems.

``` console
# mount /home
# time rsync -aHAXvx /home/ /mnt/sata_temp/

real    5m29.869s
user    0m15.302s
sys     0m59.440s
```

At this point the `/home` partition **should never be used again**.

To make sure no files are written to it; unmount it, disable its entry in `/etc/fstab`
and **reboot**.

!!! warning "Do not try to reformat the NVMe partition without rebooting. It won't work."

#### Copy SATA to NVMe SSD

After rebooting without mounting the (old) `/home` partition, create a new XFS file
system in the NVME partition and take note of its new block ID:

``` console
# time mkfs.xfs -f /dev/nvme0n1p5
meta-data=/dev/nvme0n1p5         isize=512    agcount=4, agsize=232323200 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=929292800, imaxpct=5
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=453756, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.

real    0m21.188s
user    0m0.010s
sys     0m0.103s

# blkid /dev/nvme0n1p5
/dev/nvme0n1p5: UUID="1a2f94cc-315f-4bf1-8b59-0575b49fe098" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="df113399-0802-42bf-b170-1a9e62b79220"

# blkid /dev/sda
/dev/sda: UUID="76347c52-c635-49b3-baa3-4ea98e41b4a4" BLOCK_SIZE="4096" TYPE="xfs"
```

Update the `/etc/fstab` entries for `/home` and `/home/ssd` with their new Block ID
(and `xfs` instead of `btrfs`), then mount **only** `/home` using its `/etc/fstab`
entry, mount the SATA SSD in the temporary directory (so it's not under `/home`) and
copy all its data back:

``` console
# mount /home
# mount /dev/sda /mnt/sata_temp
# time rsync -aHAXvx /mnt/sata_temp/ /home/

sent 1,033,492,365,270 bytes  received 24,759,287 bytes  361,306,458.51 bytes/sec
total size is 1,041,827,909,959  speedup is 1.01

real    47m39.689s
user    12m36.162s
sys     29m17.498s

#  df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p2   60G   14G   46G  23% /
/dev/nvme0n1p4   60G   36G   25G  59% /var/lib
/dev/nvme0n1p1  1.1G  6.2M  1.1G   1% /boot/efi
/dev/nvme0n1p5  3.5T  1.1T  2.5T  30% /home
/dev/sda        3.7T  1.1T  2.7T  28% /mnt/sata_temp
```

Once all data has been copied back to the NVMe, now on a new XFS file system, and all
entries `/etc/fstab` have been updated with the new file systems' UUIDs, mount the SATA
SSD and the NAS back on their original mount points:

``` console
# umount /mnt/sata_temp
# mount /home/ssd
# mount /home/nas
# df -h
Filesystem                  Size  Used Avail Use% Mounted on
/dev/nvme0n1p2               60G   14G   46G  23% /
/dev/nvme0n1p4               60G   36G   25G  59% /var/lib
/dev/nvme0n1p1              1.1G  6.2M  1.1G   1% /boot/efi
/dev/nvme0n1p5              3.5T  1.1T  2.5T  30% /home
luggage:/volume1/NetBackup   21T   15T  6.0T  72% /home/nas
/dev/sda                    3.7T  1.1T  2.7T  28% /home/ssd
```

Finally the cluster workloads can be restored by uncordoning the node:

``` console
# kubectl uncordon octavo
node/octavo uncordoned
```

## Install Longhorn

To keep track of deployment values, create `longhorn-values.yaml` with the following
values:

!!! k8s "`longhorn-values.yaml`"

    ``` yaml
    defaultSettings:
      allowVolumeCreationWithDegradedAvailability: "true"
      createDefaultDiskLabeledNodes: "true"
      defaultDataPath: /home/longhorn
      deletingConfirmationFlag: "true"
    metrics:
      serviceMonitor:
        enabled: "true"
    ```

Before installing the Helm chart, create a `longhorn` directory under each partition
with fast local (SSD) storage:

``` console
# mkdir /home/longhorn /home/ssd/longhorn

# ls -lad /home/longhorn/ /home/ssd/longhorn/
drwxr-xr-x 2 root root 6 Jan 25 13:54 /home/longhorn/
drwxr-xr-x 2 root root 6 Jan 25 16:00 /home/ssd/longhorn/
```

[Install with Helm](https://longhorn.io/docs/1.10.1/deploy/install/install-with-helm/)
using the latest of **Chart versions** available in
[artifacthub.io/longhorn](https://artifacthub.io/packages/helm/longhorn/longhorn):

``` console
$ helm repo add longhorn https://charts.longhorn.io
"longhorn" has been added to your repositories

$ helm repo update
from your chart repositories...
...Successfully got an update from the "longhorn" chart repository
Update Complete. ⎈Happy Helming!⎈

$ helm upgrade --install longhorn longhorn/longhorn \
  --values longhorn-values.yaml \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.10.1
Release "longhorn" does not exist. Installing it now.
I0125 14:20:26.605248  523341 warnings.go:107] "Warning: unrecognized format \"int64\""
I0125 14:20:26.606712  523341 warnings.go:107] "Warning: unrecognized format \"int64\""
I0125 14:20:26.610027  523341 warnings.go:107] "Warning: unrecognized format \"int64\""
I0125 14:20:26.610080  523341 warnings.go:107] "Warning: unrecognized format \"int64\""
NAME: longhorn
LAST DEPLOYED: Sun Jan 25 14:20:26 2026
NAMESPACE: longhorn-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Longhorn is now installed on the cluster!

Please wait a few minutes for other Longhorn components such as CSI deployments, Engine Images, and Instance Managers to be initialized.

Visit our documentation at https://longhorn.io/docs/
```

After a few minutes all the pods and services are up and running:

??? terminal "`kubectl get all -n longhorn-system`"

    ```
    $ kubectl get all -n longhorn-system
    NAME                                                    READY   STATUS    RESTARTS   AGE
    pod/csi-attacher-5857549d6f-57tz9                       1/1     Running   0          4m4s
    pod/csi-attacher-5857549d6f-cd5l8                       1/1     Running   0          4m4s
    pod/csi-attacher-5857549d6f-nh7dn                       1/1     Running   0          4m4s
    pod/csi-provisioner-57f9d44448-6g5kf                    1/1     Running   0          4m3s
    pod/csi-provisioner-57f9d44448-b2p59                    1/1     Running   0          4m4s
    pod/csi-provisioner-57f9d44448-zp4wd                    1/1     Running   0          4m3s
    pod/csi-resizer-547f8b9dc8-2nw8d                        1/1     Running   0          4m3s
    pod/csi-resizer-547f8b9dc8-sq7kx                        1/1     Running   0          4m4s
    pod/csi-resizer-547f8b9dc8-t9mff                        1/1     Running   0          4m4s
    pod/csi-snapshotter-8558df8679-2s7vw                    1/1     Running   0          4m3s
    pod/csi-snapshotter-8558df8679-7cfmt                    1/1     Running   0          4m3s
    pod/csi-snapshotter-8558df8679-9st8q                    1/1     Running   0          4m3s
    pod/engine-image-ei-3154f3aa-4p885                      1/1     Running   0          4m9s
    pod/instance-manager-106c7c23639743eccb1b438e18f1bc72   1/1     Running   0          4m9s
    pod/longhorn-csi-plugin-js5zc                           3/3     Running   0          4m3s
    pod/longhorn-driver-deployer-676f7f6c5c-v4kvh           1/1     Running   0          4m17s
    pod/longhorn-manager-7446v                              2/2     Running   0          4m17s
    pod/longhorn-ui-7c54575f4d-8ccrr                        1/1     Running   0          4m17s
    pod/longhorn-ui-7c54575f4d-pfghw                        1/1     Running   0          4m18s

    NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    service/longhorn-admission-webhook   ClusterIP   10.104.236.167   <none>        9502/TCP   4m19s
    service/longhorn-backend             ClusterIP   10.97.214.18     <none>        9500/TCP   4m19s
    service/longhorn-frontend            ClusterIP   10.108.65.154    <none>        80/TCP     4m19s
    service/longhorn-recovery-backend    ClusterIP   10.101.82.196    <none>        9503/TCP   4m19s

    NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    daemonset.apps/engine-image-ei-3154f3aa   1         1         1       1            1           <none>          4m9s
    daemonset.apps/longhorn-csi-plugin        1         1         1       1            1           <none>          4m4s
    daemonset.apps/longhorn-manager           1         1         1       1            1           <none>          4m19s

    NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/csi-attacher               3/3     3            3           4m4s
    deployment.apps/csi-provisioner            3/3     3            3           4m4s
    deployment.apps/csi-resizer                3/3     3            3           4m4s
    deployment.apps/csi-snapshotter            3/3     3            3           4m4s
    deployment.apps/longhorn-driver-deployer   1/1     1            1           4m19s
    deployment.apps/longhorn-ui                2/2     2            2           4m19s

    NAME                                                  DESIRED   CURRENT   READY   AGE
    replicaset.apps/csi-attacher-5857549d6f               3         3         3       4m4s
    replicaset.apps/csi-provisioner-57f9d44448            3         3         3       4m4s
    replicaset.apps/csi-resizer-547f8b9dc8                3         3         3       4m4s
    replicaset.apps/csi-snapshotter-8558df8679            3         3         3       4m4s
    replicaset.apps/longhorn-driver-deployer-676f7f6c5c   1         1         1       4m19s
    replicaset.apps/longhorn-ui-7c54575f4d                2         2         2       4m19s
    ```

### Longhorn UI Ingress

To make the Longhorn UI, create an `Ingress` to access it over HTTPS:

!!! k8s "`pomerium/pomerium-ingress/longhorn.yaml`"

    ``` yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: longhorn-pomerium-ingress
      namespace: longhorn-system
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        ingress.pomerium.io/pass_identity_headers: true
        ingress.pomerium.io/preserve_host_header: true
    spec:
      ingressClassName: pomerium
      rules:
        - host: longhorn.very-very-dark-gray.top
          http:
            paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: longhorn-frontend
                  port:
                    name: "http"
      tls:
        - hosts:
            - longhorn.very-very-dark-gray.top
          secretName: tls-secret
    ```

``` console hl_lines="9"
$ kubectl apply -k pomerium/pomerium-ingress/
ingress.networking.k8s.io/audiobookshelf-pomerium-ingress unchanged
ingress.networking.k8s.io/code-server-pomerium-ingress unchanged
ingress.networking.k8s.io/firefly-iii-pomerium-ingress unchanged
ingress.networking.k8s.io/home-assistant-pomerium-ingress unchanged
ingress.networking.k8s.io/homepage-pomerium-ingress unchanged
ingress.networking.k8s.io/komga-pomerium-ingress unchanged
ingress.networking.k8s.io/dashboard-pomerium-ingress unchanged
ingress.networking.k8s.io/longhorn-pomerium-ingress created
ingress.networking.k8s.io/jellyfin-pomerium-ingress unchanged
ingress.networking.k8s.io/grafana-pomerium-ingress unchanged
ingress.networking.k8s.io/influxdb-pomerium-ingress unchanged
ingress.networking.k8s.io/prometheus-pomerium-ingress unchanged
ingress.networking.k8s.io/navidrome-pomerium-ingress unchanged
ingress.networking.k8s.io/ryot-pomerium-ingress unchanged
ingress.networking.k8s.io/steam-headless-pomerium-ingress unchanged
ingress.networking.k8s.io/unifi-network-app-pomerium-ingress unchanged
ingress.networking.k8s.io/ddns-updater-pomerium-ingress unchanged
```

After a little over a minute the DNS challenge is completed and the Longhorn UI is
live at [longhorn.very-very-dark-gray.top](https://longhorn.very-very-dark-gray.top)

``` console
$ kubectl get challenge -n longhorn-system --watch
NAME                                STATE     DOMAIN                             AGE
tls-secret-1-2700837206-765358469   pending   longhorn.very-very-dark-gray.top   7s
tls-secret-1-2700837206-765358469   valid     longhorn.very-very-dark-gray.top   79s

$ kubectl get certificate -n longhorn-system
NAME         READY   SECRET       AGE
tls-secret   True    tls-secret   89s
```

At first the UI will only show that is **1 Node** and it is **Disabled**:

![](../media/2026-01-25-migrating-kubernetes-volumes-to-longhorn/longhorn-node-disabled.png)

To make the node available, add to it
[the `create-default-disk` label](https://longhorn.io/docs/1.10.1/nodes-and-volumes/nodes/default-disk-and-node-config/):

``` console
$ kubectl label node octavo node.longhorn.io/create-default-disk=true
node/octavo labeled
```
Once the node is labeled it becomes **Schedulable** and the next steps can be taken.

![](../media/2026-01-25-migrating-kubernetes-volumes-to-longhorn/longhorn-node-schedulable.png)

### Configure disks

One the node is **Schedulable** go to the **Nodes** tab and use the drop-down menu on
the right end of the node's entry to **Edit node and disks**. Add the SATA SSD, add a
**+New Disk Tag** to each disk to reflect their hardware interface (and bandwidth) and
set their **Storage Reserved** to the recommended **50 Gi**.

![](../media/2026-01-25-migrating-kubernetes-volumes-to-longhorn/longhorn-node-2-disks-ready.png)

Storage metrics represent aggregate values across all disks configured on all nodes:

*   **Reserved storage** is the space Longhorn **will not use** for volume replicas.
    It is set aside for the Host OS, other applications, and to prevent the disk from
    reaching 100% capacity.
*   **Used storage** is the **actual physical space** currently occupied by Longhorn
    data and other system files.
*   **Schedulable storage** is the amount of **new volume capacity** Longhorn can
    allocate to pods.

### Targeted `StorageClass`es

To enable the automatic creation of volumes in each disk, create a `StorageClass` for
each type of disk (NVMe, SATA) with this manifest:

!!! k8s "`longhorn-storage.yaml`"

    ``` yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: longhorn-nvme
    provisioner: driver.longhorn.io
    allowVolumeExpansion: true
    parameters:
      numberOfReplicas: "1" # Increase to 2 after adding a second node
      diskSelector: "nvme"
      dataLocality: "best-effort"
    ---
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: longhorn-sata
    provisioner: driver.longhorn.io
    allowVolumeExpansion: true
    parameters:
      numberOfReplicas: "1"
      diskSelector: "sata"
      dataLocality: "best-effort"
    ```

Longhorn supports supports online volume expansion, allowing the volume to grow without
downtime, **only if** the `StorageClass` has `allowVolumeExpansion: true` which cannot
be added later without deleting and recreating the `StorageClass` (existing volumes are
not deleted).

``` console
$ kubectl apply -f longhorn-storage.yaml
storageclass.storage.k8s.io/longhorn-nvme created
storageclass.storage.k8s.io/longhorn-sata created
```

### Replication Vs. Bandwidth

When creating new PVC using the `longhorn-nvme` class to use the fastest SSD,
`accessModes` should *almost always* be set to `ReadWriteOnce`.

**RWO** volumes (`ReadWriteOnce`) are mounted directly as block devices via iSCSI. This
provides the highest performance for database-like workloads (e.g., Postgres, Redis, or
application caches) because there is no network filesystem overhead. Most Kubernetes
deployments (even those with multiple replicas) do not require
**multiple pods to write to the same volume simultaneously**.
Instead, each pod typically manages its own data. In a multi-node cluster, if one node
fails, Kubernetes will move the pod to the other node. Longhorn will then detach the RWO
cvolume from the old node and attach it to the new one.

**RWX** volumes (`ReadWriteMany`) should only be used when an application specifically
requires **multiple pods** (possibly on multiple nodes) to 
**read and write to the exact same files at the same time**. 
Longhorn implements RWX by spinning up a "Share Manager" pod that acts as an NFS server
for that specific volume. This mode requires the `nfs-common` package you installed on
your Ubuntu hosts earlier.

When scaling a single-node cluster to two nodes and setting `numberOfReplicas: 2`,
an RWO volume will still be fully distributed and synced across both your i7 and i5 NVMe
SSDs. The "Once" in `ReadWriteOnce` refers only to *how many nodes can mount the volume*
at one time, not how many nodes store the data. Even with RWO, your data is redundant
and safe on both nodes.

However, in a two-node Longhorn cluster with two replicas, each pod will **not** use *exclusively local PCIe NVMe bandwidth* for its storage operations. While each pod can
be guaranteed to have a local copy of its data, the synchronous replication requirement
of a distributed system introduces network latency. 

By default, Longhorn may schedule a pod on a node that does not contain a local replica
of its volume. To force each pod to prioritize its local disk, enable **Data Locality**
in the `StorageClass` or volume settings: 

*   `best-effort`: Longhorn attempts to keep one replica on the same node as the pod.
    If it can't, the pod will still run but will access its data over the network from
    the other node.
*   `strict-local`: Enforces that a healthy replica must exist on the same node as the
    pod. This provides the lowest possible latency for reads but prevents the pod from
    starting if the local disk is unavailable. 

Even if a pod is reading directly from its local NVMe SSD, write operations are
synchronous. When a pod writes data, the Longhorn Engine (running on the same node as
the pod) must successfully write that data to both the local replica and the remote
replica on the second node before the write is considered "complete".
This means the write bandwidth and latency are capped by the network speed and the
overhead of the iSCSI/Longhorn protocol, rather than the raw PCIe NVMe bandwidth.

While the pod will benefit from NVMe speeds for local reads (if data locality is
enabled), the overall performance will be significantly lower than a raw local NVMe SSD:
reads will have *near-local* speeds if a local replica is present, but writes will be
limited by network latency and the processing time of the Longhorn engine.

Ultimately, if an application requires raw PCIe NVMe bandwidth and doesn't need
Longhorn’s high availability features (cross-node syncing, et.c.), then a `hostPath` or
Local Path Provisioner class may be best for that specific workload. However, for most
general-purpose applications, the performance trade-off of Longhorn is acceptable for
the benefit of having a fully synced, distributed cluster.

### Active-Active Vs. Active-Passive

For a single-pod deployment like code-server, when scaling up to 2 replicas on 2 nodes,
with Longhorn replicating its RWO volume with **data locality** set to `best-effort` to
keep both volumes in sync, the result is an **active-passive** with one pod being
active while the other stays in stand-by to take over only when the first one goes down.

This setup will not work for an **active-active** setup because of two reasons:

1.  A `ReadWriteOnce` (RWO) volume is **physically locked to a single** node at a time.
    If Pod-A is running one node and has the volume mounted, Pod-B on the other node
    **will be unable to start**. It will stay in a `ContainerCreating` or
    `MatchNodeSelector` state because Longhorn cannot attach an RWO volume to two nodes
    simultaneously. While Longhorn replicates the data to both nodes' NVMe SSDs in the
    background, only one engine can be the **Primary** (the writer) at any given moment.

    With `dataLocality: best-effort` and `numberOfReplicas: 2` every byte Pod-A writes
    to the one NVMe is synchronously sent over the network to the other NVMe, so that
    both disks are always bit-for-bit identical. If one node crashes, Kubernetes
    detects the node failure. It then schedules a new Pod on the other node. Longhorn
    "promotes" the second replica to be the new **Primary**, and the Pod starts.
    This is an **Active-Passive HA** setup, with redundancy, but not both pods
    responding to requests at the same time.

2.  Even when using **RWX** (which allows both pods to run), applications like
    `code-server` are not **stateless**; `code-server` (and its underlying VS Code
    engine) uses **SQLite** databases for extensions and settings. SQLite does not
    support multiple processes writing to the same file over a network (NFS/RWX).
    Attempting to run multiple instances on a single such database would lead to
    database corruption or immediate "Locked" errors. Moreover, if Pomerium proxy sends
    a request for "Save File" to Pod-B, but the "Open Editor" session was handled by
    Pod-A, the session state would be inconsistent.

RWO volumes (best compromise for speed and replication) support **Disaster Recovery** (Active-Passive) setups, ensuring data is never lost a node dies. However, it cannot be
Active-Active; for that, an application must be specifically designed to store its
state in an external database (like Postgres) rather than a local RWO/RWX volume.

Most of the currently running applications are designed as monolithic services that rely
on a single local database (SQLite) or a local file system (e.g. Home Assistant), making
them **Active-Passive** by nature. However, several can be adapted for Active-Active HA
with specific configurations: Pomerium and Homepage are stateless, Grafana can have its
SQLite database replaced by Postgres or MySQL, but ony InfluxDB 3.0 supports clustering.

## From `hostPath` to Longhorn

At this point data can be migrated from the old `hostPath` volume to the new Longhorn
volumes in three steps for each application (**Deployment**):

1.  Create a new `PersistentVolumeClaim` using `storageClassName: longhorn-nvme`.
2.  Scale the deployment down.
3.  Copy data using a simple pod to run the `cp` command e.g. using `busybox`.
4.  Update the Deployment to mount the new `PersistentVolumeClaim`.
5.  Scale the deployment back up.

To copy the data the same simple pod can be run by adjusting just the highlighted values:

!!! k8s "`longhorn-migrator.yaml`"

    ``` yaml hl_lines="5 27 30"
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: hostpath-to-longhorn-migrator
      namespace: code-server  # Set to each application's namespace
    spec:
      template:
        spec:
          restartPolicy: OnFailure  # Restart only upon failure.
          containers:
          - name: worker
            image: ubuntu:22.04
            command: ["/bin/sh", "-c"]
            args:
              - |
                apt-get update && apt-get install -y rsync
                rsync -uva /old/ /new/
            volumeMounts:
            - name: old-data
              mountPath: /old  # Point to existing hostPath subdirectory
              readOnly: true
            - name: new-data
              mountPath: /new  # Point to new Longhorn PVC
          volumes:
          - name: old-data
            hostPath:
              path: /home/k8s/code-server
          - name: new-data
            persistentVolumeClaim:
              claimName: code-server-pvc-lh  # The new PVC created for each application.
    ```

### Example migration

To illustrate the migration process with a simple case, start by migrating
[VS Code Server](./2023-05-29-running-vs-code-server-on-kubernetes.md).

1.  Update the `code-server.yaml` manifest to add a new `PersistentVolumeClaim` using
    `storageClassName: longhorn-nvme` and apply the mani fest to create the PVC:

    !!! k8s "`code-server.yaml`"

        ``` yaml linenums="47" hl_lines="4 5 7 9"
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: code-server-pvc-lh
          namespace: code-server
        spec:
          storageClassName: longhorn-nvme
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
        ```

    !!! warning
    
        The `accessModes` value **cannot** be *easly* done later, e.g. when [adding a
        node](#adding-future-nodes); see
        [Replication Vs. Bandwidth](#replication-vs-bandwidth).

    Confirm the PVC is created after applying the `code-server.yaml` manifest:
    
    ``` console hl_lines="6 12"
    $ kubectl apply -f code-server.yaml 
    namespace/code-server unchanged
    service/code-server unchanged
    persistentvolume/code-server-pv unchanged
    persistentvolumeclaim/code-server-pv-claim unchanged
    persistentvolumeclaim/code-server-pvc-lh created
    deployment.apps/code-server unchanged

    $ kubectl -n code-server get pvc
    NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
    code-server-pv-claim   Bound    code-server-pv                             10Gi       RWO            manual          <unset>                 269d
    code-server-pvc-lh     Bound    pvc-f549ff26-5424-4b30-b0a3-245a845888f7   5Gi        RWO            longhorn-nvme   <unset>                 68s
    ```

2.  Scale the deployment down.

    ``` console
    $ kubectl -n code-server scale deployment code-server --replicas=0
    deployment.apps/code-server scaled
    ```

3.  Copy data using a simple pod to run the `cp` command, e.g. using `busybox`.

    !!! k8s "`longhorn-migrator.yaml`"

        ``` yaml hl_lines="5 27 30"
        apiVersion: batch/v1
        kind: Job
        metadata:
          name: hostpath-to-longhorn-migrator
          namespace: code-server  # Set to each application's namespace
        spec:
          template:
            spec:
              restartPolicy: OnFailure  # Restart only upon failure.
              containers:
              - name: worker
                image: ubuntu:22.04
                command: ["/bin/sh", "-c"]
                args:
                  - |
                    apt-get update && apt-get install -y rsync
                    rsync -uva /old/ /new/
                volumeMounts:
                - name: old-data
                  mountPath: /old  # Point to existing hostPath subdirectory
                  readOnly: true
                - name: new-data
                  mountPath: /new  # Point to new Longhorn PVC
              volumes:
              - name: old-data
                hostPath:
                  path: /home/k8s/code-server
              - name: new-data
                persistentVolumeClaim:
                  claimName: code-server-pvc-lh  # The new PVC created for each application.
        ```

    ``` console
    $ kubectl apply -f longhorn-migrator.yaml 
    job.batch/hostpath-to-longhorn-migrator created

    $ kubectl -n code-server get jobs --watch
    NAME                            STATUS    COMPLETIONS   DURATION   AGE
    hostpath-to-longhorn-migrator   Running   0/1           12s        12s
    hostpath-to-longhorn-migrator   Running   0/1           18s        18s
    hostpath-to-longhorn-migrator   SuccessCriteriaMet   0/1           19s        19s
    hostpath-to-longhorn-migrator   Complete             1/1           19s        19s

    $ kubectl -n code-server delete job hostpath-to-longhorn-migrator 
    job.batch "hostpath-to-longhorn-migrator" deleted from code-server namespace
    ```

4.  Update the Deployment to mount the new `PersistentVolumeClaim`.

    !!! k8s "`code-server.yaml`"

        ``` yaml linenums="76" hl_lines="4"
        volumes:
          - name: code-server-storage
            persistentVolumeClaim:
              claimName: code-server-pvc-lh
        ```
    
    At this point the old `PersistentVolumeClaim` and `PersistentVolume` using
    `hostPath` can be removed. Apply the `code-server.yaml` manifest again:
    
    ``` console
    $ kubectl apply -f code-server.yaml 
    namespace/code-server unchanged
    service/code-server unchanged
    persistentvolumeclaim/code-server-pvc-lh unchanged
    deployment.apps/code-server configured
    ```

5.  Scale the deployment back up.

    ``` console
    $ kubectl -n code-server scale deployment code-server --replicas=1
    deployment.apps/code-server scaled

    $ kubectl -n code-server get pods --watch
    NAME                           READY   STATUS              RESTARTS   AGE
    code-server-67c85cf5d7-gwpnk   0/1     ContainerCreating   0          1s
    code-server-67c85cf5d7-gwpnk   1/1     Running             0          12s
    ```

6.  Clean-up the old `PersistentVolumeClaim` and `PersistentVolume` based on `hostPath`:

    ``` console
    $ kubectl -n code-server delete pvc code-server-pv-claim
    persistentvolumeclaim "code-server-pv-claim" deleted from code-server namespace

    $ kubectl delete pv code-server-pv
    persistentvolume "code-server-pv" deleted
    ```

## Longhorn Backups

Once pods are migrated to Longhorn volumes, setting up backups to the Synology NAS is
*easy*.

First, create the necessary directories in the NAS:

``` console
# mkdir /home/nas/backups/longhorn
```

Then, **Edit** the **default** target under **Backup and Restore > Backup Targets** and
set the URL to `nfs://192.168.0.4:/volume1/NetBackup/backups/longhorn` **after** having
creating the directories in the NAS.

### Recurrent jobs

To set up a global backup system that covers volumes across all namespaces, leverage
[Longhorn’s Recurring Job Groups](https://longhorn.io/docs/1.10.1/snapshots-and-backups/scheduling-backups-and-snapshots/).
These allow defining the schedule once and then applying it to any PVC simply by adding
a label.

Create the Global Recurring Jobs with the following `longhorn-backups.yaml` manifest
to create the jobs in the `longhorn-system` namespace. By adding them to the **default**
group, they become available to any volume in the cluster.


!!! k8s "`longhorn-backups.yaml`"

    ``` yaml
    apiVersion: longhorn.io/v1beta2
    kind: RecurringJob
    metadata:
      name: global-daily-backup
      namespace: longhorn-system
    spec:
      cron: "0 2 * * *"        # 2:00 AM daily
      task: "backup"           
      groups:
      - default                # Group name used for assignment
      retain: 7                # Keeps 1 week of dailies
      concurrency: 2           # Allows 2 volumes to backup simultaneously
    ---
    apiVersion: longhorn.io/v1beta2
    kind: RecurringJob
    metadata:
      name: global-weekly-backup
      namespace: longhorn-system
    spec:
      cron: "0 3 * * 0"        # 3:00 AM every Sunday
      task: "backup"
      groups:
      - default
      retain: 4                # Keeps 1 month of weeklies
      concurrency: 1
    ```

``` console
$ kubectl apply -f longhorn-backups.yaml
recurringjob.longhorn.io/global-daily-backup created
recurringjob.longhorn.io/global-weekly-backup created
```

In Longhorn, jobs do not target specific namespaces; instead, Volumes (the underlying
objects of PVCs) "subscribe" to Groups. When a volume has a label matching a group name
defined in a `RecurringJob`, Longhorn automatically includes that volume in the
schedule. Because Longhorn volumes are cluster-scoped, this works regardless of which
namespace the PVC resides in.

Add the following label to the manifest of each PVC to be backed up. Longhorn will
automatically propagate this label to the underlying volume.

!!! k8s "`code-server.yaml`"

    ``` yaml linenums="20" hl_lines="6"
    metadata:
      name: code-server-pvc-lh
      namespace: code-server
      labels:
        # This enables all jobs in the 'default' group for this volume
        recurring-job-group.longhorn.io/default: "enabled"
    ```

Backups will be found under **Backup and Restore > Backups** after the first 2:00 AM
run, identified by their source Volume Name and Timestamp, regardless of their original
Kubernetes namespace.

## Adding future nodes

To add nodes in the future (TBC):

1.  Create an `XFS` (or `ext4`) file system and mount it on `/home`.
1.  Create the `/home/longhorn` directory.
1.  Join the node to the cluster.
1.  Label the node with `node.longhorn.io/create-default-disk=true`.
    *   Longhorn will automatically detect `/home/longhorn` on the new node.
1.  Update the `longhorn-nvme` `StorageClass`. or individual volumes in the relevant
    deployments, to `numberOfReplicas: 2` so that Longhorn syncs data across nodes.
