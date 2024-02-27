---
title:  "Audiobookshelf on Kubernetes"
date:   2024-02-28 02:28:24 +0200
categories: linux kubernetes audiobookshelf docker server
---

[Migrating a Plex Media Server to Kubernetes](https://stibbons1990.github.io/hex/2023/09/16/migrating-a-plex-media-server-to-kubernetes.html),
was a significant improvement for the *maintenance* of the Plex Media
Server I use to listen to podcasts and audiobooks, to keep me company
while I play games, but after all these years Plex remains a
*very insufficient and deficient* application for audiobooks.

***Enter [audiobookshelf](https://www.audiobookshelf.org/)*** (because Emby and Jellyfin are also not great)

## Installation on Kubernetes

Audiobookshelf
[configuration](https://www.audiobookshelf.org/docs/#env-configuration)
requires writeable directories mounted at
`/config` and `/metadata` for database, cache, etc.

```
# useradd -d /home/k8s/audiobookshelf -s /usr/sbin/nologin audiobookshelf
# mkdir /home/k8s/audiobookshelf/config /home/k8s/audiobookshelf/metadata
# chown -R audiobookshelf.audiobookshelf /home/k8s/audiobookshelf
# ls -dln /home/k8s/audiobookshelf/
drwxr-xr-x 1 1006 1006 28 Feb 27 22:47 /home/k8s/audiobookshelf/
```

Note the UID/GID (**1006**) to be used in the Kubernetes
deployment `securityContext` later.

[Docker Compose](https://www.audiobookshelf.org/docs/#docker-compose-install)
suggests mounting audiobooks and podcasts as separate
volumes so the following Kubernetes deployment will do
so, even though it would also work to have it all under
a single volume.

Create the following `audiobookshelf.yaml` and deploy
it:

```
$ kubectl apply -f audiobookshelf.yaml
namespace/audiobookshelf created
persistentvolume/audiobookshelf-pv-config created
persistentvolume/audiobookshelf-pv-metadata created
persistentvolume/audiobookshelf-pv-audiobooks created
persistentvolume/audiobookshelf-pv-podcasts created
persistentvolumeclaim/audiobookshelf-pvc-config created
persistentvolumeclaim/audiobookshelf-pvc-metadata created
persistentvolumeclaim/audiobookshelf-pvc-audiobooks created
persistentvolumeclaim/audiobookshelf-pvc-podcasts created
service/audiobookshelf-tcp created
deployment.apps/audiobookshelf created
ingress.networking.k8s.io/audiobookshelf-ingress created

$ $ kubectl -n audiobookshelf get service
NAME                        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
service/audiobookshelf-tcp   NodePort   10.97.42.11   <none>        13378:31378/TCP   51m
cm-acme-http-solver-jh67q   NodePort   10.106.142.67   <none>        8089:30126/TCP    2m39s
```

The server is now available at http://192.168.0.6:31378

NGinx should also make it available at https://aus.ssl.uu.am


### Deployment

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: audiobookshelf
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: audiobookshelf-pv-config
  namespace: audiobookshelf
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/k8s/audiobookshelf/config
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: audiobookshelf-pv-metadata
  namespace: audiobookshelf
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/k8s/audiobookshelf/metadata
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: audiobookshelf-pv-audiobooks
  namespace: audiobookshelf
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/depot/audio/Audiobooks
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: audiobookshelf-pv-podcasts
  namespace: audiobookshelf
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/depot/audio/Podcasts
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-pvc-config
  namespace: audiobookshelf
spec:
  storageClassName: manual
  volumeName: audiobookshelf-pv-config
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-pvc-metadata
  namespace: audiobookshelf
spec:
  storageClassName: manual
  volumeName: audiobookshelf-pv-metadata
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-pvc-audiobooks
  namespace: audiobookshelf
spec:
  storageClassName: manual
  volumeName: audiobookshelf-pv-audiobooks
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-pvc-podcasts
  namespace: audiobookshelf
spec:
  storageClassName: manual
  volumeName: audiobookshelf-pv-podcasts
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: audiobookshelf
  name: audiobookshelf
  namespace: audiobookshelf
spec:
  replicas: 1
  revisionHistoryLimit: 0
  selector:
    matchLabels:
      app: audiobookshelf
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: audiobookshelf
    spec:
      containers:
        - image: ghcr.io/advplyr/audiobookshelf:latest
          imagePullPolicy: Always
          name: audiobookshelf
          env:
          - name: PORT
            value: "13378"
          ports:
          - containerPort: 13378
          resources: {}
          stdin: true
          tty: true
          volumeMounts:
          - mountPath: /config
            name: audiobookshelf-config
          - mountPath: /metadata
            name: audiobookshelf-metadata
          - mountPath: /audiobooks
            name: audiobookshelf-audiobooks
          - mountPath: /podcasts
            name: audiobookshelf-podcasts
          securityContext:
            allowPrivilegeEscalation: false
            runAsUser: 1006
            runAsGroup: 1006
      restartPolicy: Always
      volumes:
      - name: audiobookshelf-config
        persistentVolumeClaim:
          claimName: audiobookshelf-pvc-config
      - name: audiobookshelf-metadata
        persistentVolumeClaim:
          claimName: audiobookshelf-pvc-metadata
      - name: audiobookshelf-audiobooks
        persistentVolumeClaim:
          claimName: audiobookshelf-pvc-audiobooks
      - name: audiobookshelf-podcasts
        persistentVolumeClaim:
          claimName: audiobookshelf-pvc-podcasts
---
kind: Service
apiVersion: v1
metadata:
  name: audiobookshelf-tcp
  namespace: audiobookshelf
spec:
  type: NodePort
  ports:                      
  - port: 13378
    nodePort: 31378
    targetPort: 13378
  selector:
    app: audiobookshelf

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: audiobookshelf-ingress
  namespace: audiobookshelf
  annotations:
    acme.cert-manager.io/http01-edit-in-place: "true"
    cert-manager.io/issue-temporary-certificate: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
    - host: aus.ssl.uu.am
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: audiobookshelf
                port:
                  number: 31378
  tls:
    - secretName: tls-secret
      hosts:
        - aus.ssl.uu.am
```

The above deployment is based on
https://www.audiobookshelf.org/docs/#docker-upgrade
and
https://www.datree.io/helm-chart/audiobookshelf-truecharts


#### Troubleshooting

Contrary to what
[Configuration](https://www.audiobookshelf.org/docs/#configuration)
would suggest, the server will by default try to listen on port 80.
To override this the `env` variable `PORT` must be set:

```yaml
          env:
          - name: PORT
            value: "13378"
```

Otherwise, when running as a non-privileged user, this will cause it to crash-loop:

```
$ kubectl get all -n audiobookshelf
NAME                                  READY   STATUS             RESTARTS      AGE
pod/audiobookshelf-754d55cd68-z4br6   0/1     CrashLoopBackOff   4 (59s ago)   3m9s

NAME                         TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)           AGE
service/audiobookshelf-tcp   NodePort   10.97.42.11   <none>        13378:32164/TCP   10m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/audiobookshelf   0/1     1            0           3m9s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/audiobookshelf-754d55cd68   1         1         0       3m9s

$ kubectl -n audiobookshelf describe pod audiobookshelf-754d55cd68-z4br6
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  5m56s                  default-scheduler  Successfully assigned audiobookshelf/audiobookshelf-754d55cd68-z4br6 to lexicon
  Normal   Pulled     5m29s                  kubelet            Successfully pulled image "ghcr.io/advplyr/audiobookshelf:latest" in 25.769s (25.769s including waiting)
  Normal   Pulled     5m26s                  kubelet            Successfully pulled image "ghcr.io/advplyr/audiobookshelf:latest" in 661ms (661ms including waiting)
  Normal   Pulled     5m9s                   kubelet            Successfully pulled image "ghcr.io/advplyr/audiobookshelf:latest" in 659ms (659ms including waiting)
  Normal   Created    4m39s (x4 over 5m29s)  kubelet            Created container audiobookshelf
  Normal   Started    4m39s (x4 over 5m29s)  kubelet            Started container audiobookshelf
  Normal   Pulled     4m39s                  kubelet            Successfully pulled image "ghcr.io/advplyr/audiobookshelf:latest" in 752ms (752ms including waiting)
  Normal   Pulling    3m48s (x5 over 5m55s)  kubelet            Pulling image "ghcr.io/advplyr/audiobookshelf:latest"
  Normal   Pulled     3m47s                  kubelet            Successfully pulled image "ghcr.io/advplyr/audiobookshelf:latest" in 634ms (634ms including waiting)
  Warning  BackOff    47s (x22 over 5m25s)   kubelet            Back-off restarting failed container audiobookshelf in pod audiobookshelf-754d55cd68-z4br6_audiobookshelf(147c22bc-5f0b-4ed8-a881-478748234776)

$ kubectl -n audiobookshelf logs audiobookshelf-754d55cd68-z4br6
Config /config /metadata
[2024-02-27 22:29:14.925] INFO: === Starting Server ===
[2024-02-27 22:29:14.939] INFO: [Server] Init v2.8.0
[2024-02-27 22:29:14.967] INFO: [Database] Initializing db at "/config/absdatabase.sqlite"
[2024-02-27 22:29:14.998] INFO: [Database] Db connection was successful
[2024-02-27 22:29:15.065] INFO: [Database] Db initialized with models: user, library, libraryFolder, book, podcast, podcastEpisode, libraryItem, mediaProgress, series, bookSeries, author, bookAuthor, collection, collectionBook, playlist, playlistMediaItem, device, playbackSession, feed, feedEpisode, setting, customMetadataProvider
[2024-02-27 22:29:15.080] INFO: [LogManager] Init current daily log filename: 2024-02-27.txt
[2024-02-27 22:29:15.084] INFO: [BackupManager] 0 Backups Found
[2024-02-27 22:29:15.085] INFO: [BackupManager] Auto Backups are disabled
Warning: connect.session() MemoryStore is not
designed for a production environment, as it will leak
memory, and will not scale past a single process.
[2024-02-27 22:29:15.095] FATAL: [Server] Uncaught exception origin: uncaughtException, error: Error: listen EACCES: permission denied 0.0.0.0:80
    at Server.setupListenHandle [as _listen2] (node:net:1855:21)
    at listenInCluster (node:net:1920:12)
    at Server.listen (node:net:2008:7)
    at Server.start (/server/Server.js:319:17) {
  code: 'EACCES',
  errno: -13,
  syscall: 'listen',
  address: '0.0.0.0',
  port: 80
} (Server.js:160)
node:events:496
      throw er; // Unhandled 'error' event
      ^

Error: listen EACCES: permission denied 0.0.0.0:80
    at Server.setupListenHandle [as _listen2] (node:net:1855:21)
    at listenInCluster (node:net:1920:12)
    at Server.listen (node:net:2008:7)
    at Server.start (/server/Server.js:319:17)
Emitted 'error' event on Server instance at:
    at emitErrorNT (node:net:1899:8)
    at process.processTicksAndRejections (node:internal/process/task_queues:82:21) {
  code: 'EACCES',
  errno: -13,
  syscall: 'listen',
  address: '0.0.0.0',
  port: 80
}

Node.js v20.11.1
```
